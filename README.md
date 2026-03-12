# ⚙️ Symfony 6 — Clean Architecture & CQRS

CQRS (Command Query Responsibility Segregation) sépare les opérations
d'écriture (Commands) des opérations de lecture (Queries). En pratique,
sur une refonte d'application métier, ça permet d'optimiser les lectures
indépendamment des écritures, de tester chaque flux séparément, et de faire
évoluer le modèle sans tout recasser.

---

## Principe CQRS appliqué à Symfony

```
  Requête entrante
          │
          ├── Écriture (POST, PUT, DELETE)
          │       │
          │       ▼
          │   Command (objet immutable)
          │   CreateArticleCommand { title, content, authorId }
          │       │
          │       ▼
          │   CommandBus → dispatch vers Handler
          │   CreateArticleHandler::__invoke(CreateArticleCommand)
          │       │
          │       ▼
          │   Modifie le Domain, persiste via Repository
          │   Retourne : void ou ID
          │
          └── Lecture (GET)
                  │
                  ▼
              Query (objet immutable)
              GetArticleQuery { id }
                  │
                  ▼
              QueryBus → dispatch vers Handler
              GetArticleHandler::__invoke(GetArticleQuery)
                  │
                  ▼
              Retourne : DTO (pas l'entité Domain)
              ArticleDto { id, title, content, authorName, createdAt }
```

---

## Implémentation du Command Bus (Symfony Messenger)

```php
// Shared/Domain/Bus/CommandBusInterface.php
interface CommandBusInterface
{
    public function dispatch(object $command): mixed;
}

// Shared/Infrastructure/Bus/SymfonyCommandBus.php
final readonly class SymfonyCommandBus implements CommandBusInterface
{
    public function __construct(
        private MessageBusInterface $bus
    ) {}

    public function dispatch(object $command): mixed
    {
        try {
            $envelope = $this->bus->dispatch($command);
            $stamps   = $envelope->last(HandledStamp::class);
            return $stamps?->getResult();
        } catch (HandlerFailedException $e) {
            // Déballer l'exception originale (pas l'enveloppe Messenger)
            throw $e->getNestedExceptions()[0];
        }
    }
}

// config/services.yaml
services:
    App\Shared\Domain\Bus\CommandBusInterface:
        alias: App\Shared\Infrastructure\Bus\SymfonyCommandBus

    App\Shared\Domain\Bus\QueryBusInterface:
        alias: App\Shared\Infrastructure\Bus\SymfonyQueryBus
```

---

## Command + Handler — création d'un article

```php
// Article/Application/Command/CreateArticleCommand.php
final readonly class CreateArticleCommand
{
    public function __construct(
        public string $title,
        public string $content,
        public string $authorId,
    ) {}
}

// Article/Application/Handler/CreateArticleHandler.php
#[AsMessageHandler]
final readonly class CreateArticleHandler
{
    public function __construct(
        private ArticleRepositoryInterface $articles,
        private AuthorRepositoryInterface  $authors,
    ) {}

    public function __invoke(CreateArticleCommand $command): string
    {
        $author = $this->authors->findById($command->authorId)
            ?? throw new AuthorNotFoundException($command->authorId);

        $article = Article::create(
            id:      ArticleId::generate(),
            title:   $command->title,
            content: $command->content,
            author:  $author,
        );

        $this->articles->save($article);
        return $article->getId()->value();
    }
}
```

---

## Query + DTO — lecture optimisée

```php
// Article/Application/Query/GetArticlesQuery.php
final readonly class GetArticlesQuery
{
    public function __construct(
        public int    $page   = 1,
        public int    $limit  = 20,
        public string $search = '',
    ) {}
}

// Article/Application/Dto/ArticleListDto.php
// DTO de lecture : pas l'entité Domain — structure optimisée pour l'affichage
final readonly class ArticleListDto
{
    public function __construct(
        public string $id,
        public string $title,
        public string $authorName,    // dénormalisé : évite une jointure côté client
        public string $createdAtAgo,  // formattage côté serveur
        public int    $viewCount,
    ) {}
}

// Article/Application/Handler/GetArticlesHandler.php
#[AsMessageHandler]
final readonly class GetArticlesHandler
{
    public function __construct(
        private ArticleReadRepositoryInterface $readRepo
    ) {}

    public function __invoke(GetArticlesQuery $query): array
    {
        // Le read repository peut utiliser une vue SQL optimisée
        // indépendamment du write model (CQRS complet)
        return $this->readRepo->findPaginated(
            page:   $query->page,
            limit:  $query->limit,
            search: $query->search
        );
    }
}
```

---

## Controller — mince, délègue au Bus

```php
// Article/Infrastructure/Api/ArticleController.php
#[Route('/api/articles')]
final class ArticleController extends AbstractController
{
    public function __construct(
        private readonly CommandBusInterface $commands,
        private readonly QueryBusInterface   $queries,
    ) {}

    #[Route('', methods: ['GET'])]
    public function list(Request $request): JsonResponse
    {
        $articles = $this->queries->dispatch(new GetArticlesQuery(
            page:   (int) $request->query->get('page', 1),
            limit:  (int) $request->query->get('limit', 20),
            search: $request->query->get('search', ''),
        ));
        return $this->json($articles);
    }

    #[Route('', methods: ['POST'])]
    #[IsGranted('ROLE_AUTHOR')]
    public function create(Request $request): JsonResponse
    {
        $data = $request->toArray();
        $id = $this->commands->dispatch(new CreateArticleCommand(
            title:    $data['title']    ?? throw new BadRequestHttpException('title requis'),
            content:  $data['content']  ?? throw new BadRequestHttpException('content requis'),
            authorId: $this->getUser()->getId(),
        ));
        return $this->json(['id' => $id], 201);
    }
}
```

---

## Tests PHPUnit — couverture Command + Query

```php
class CreateArticleHandlerTest extends TestCase
{
    public function testCreatesArticleAndPersists(): void
    {
        $authorRepo  = $this->createMock(AuthorRepositoryInterface::class);
        $articleRepo = $this->createMock(ArticleRepositoryInterface::class);

        $author = Author::create(AuthorId::generate(), 'Test Author');
        $authorRepo->method('findById')->willReturn($author);
        $articleRepo->expects($this->once())->method('save')
            ->with($this->isInstanceOf(Article::class));

        $handler = new CreateArticleHandler($articleRepo, $authorRepo);
        $id = ($handler)(new CreateArticleCommand('Titre', 'Contenu', $author->getId()->value()));

        $this->assertNotEmpty($id);
    }

    public function testThrowsIfAuthorNotFound(): void
    {
        $authorRepo  = $this->createMock(AuthorRepositoryInterface::class);
        $articleRepo = $this->createMock(ArticleRepositoryInterface::class);
        $authorRepo->method('findById')->willReturn(null);

        $this->expectException(AuthorNotFoundException::class);
        $handler = new CreateArticleHandler($articleRepo, $authorRepo);
        ($handler)(new CreateArticleCommand('T', 'C', 'unknown-id'));
    }
}
```

---

## Ce que j'ai appris

Sur un projet de refonte, le CQRS évite un piège classique : le modèle
d'écriture (normalisé pour l'intégrité) est souvent mauvais pour la lecture
(trop de jointures, trop de champs inutiles). Avoir deux modèles distincts
permet d'optimiser chacun indépendamment — lecture via une vue SQL dénormalisée,
écriture via le Domain pur. Sur Hermes 2, cette séparation serait particulièrement
utile si l'application a des écrans de consultation fréquents et des workflows
d'écriture complexes.

---

*Projet réalisé dans le cadre de ma formation ingénieur — ENSET Mohammedia*
*Par **Abderrahmane Elouafi** · [LinkedIn](https://www.linkedin.com/in/abderrahmane-elouafi-43226736b/) · [Portfolio](https://my-first-porfolio-six.vercel.app/)*
