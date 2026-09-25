# Dev Environment – Laravel + Flux UI

Aplicação Laravel (fullstack, Livewire + Flux UI + Tailwind) rodando em Docker com PostgreSQL.

## Containers

| Container     | Função                    | Porta (host)  |
|---------------|---------------------------|---------------|
| `dev-backend` | Laravel (`artisan serve`) | 8000          |
| `dev-front`   | Vite (assets, HMR)        | 5173          |
| `dev-db`      | PostgreSQL 17             | 5433          |

## Pré-requisitos

- Docker e Docker Compose

## Como iniciar

Execute a partir da **raiz do projeto** (onde está o `docker-compose.yml`).

1. Crie os arquivos de ambiente:

   ```sh
   cp .env.example .env
   cp backend/.env.example backend/.env
   ```

   Em `.env`, ajuste `UID` e `GID` para os valores de `id -u` e `id -g`, se forem diferentes de 1000.

2. Suba os containers:

   ```sh
   docker compose up -d --build
   ```

   Na primeira vez, o `dev-backend` roda `composer install` e o `dev-front` roda `npm install`.
   Acompanhe com `docker compose logs -f` até o Laravel e o Vite estarem no ar.

3. Gere a chave da aplicação e rode as migrations:

   ```sh
   docker compose exec backend php artisan key:generate
   docker compose exec backend php artisan migrate
   ```

4. Acesse http://localhost:8000

## Comandos úteis

```sh
docker compose down                                  # parar e remover containers (mantém o banco)
docker compose down -v                               # idem, apagando também os dados do banco
docker compose exec backend php artisan <comando>    # artisan
docker compose exec backend composer <comando>       # composer
docker compose logs -f <backend|vite|db>             # logs
```

Banco de dados a partir do host: `localhost:5433` (usuário, senha e banco definidos em `.env`).

---

## Arquitetura e fluxo de uma requisição

### Visão geral

Monolito Laravel fullstack: **Livewire + Flux UI** na camada de interface, regras de negócio isoladas do framework e **PostgreSQL** para persistência. A organização segue a **Arquitetura Limpa**: as dependências apontam sempre para dentro (interface → aplicação → domínio) e o domínio não conhece Laravel, Eloquent nem HTTP.

### Como uma solicitação é processada

```
Requisição HTTP
   │
   ▼
routes/web.php | routes/api.php          → apenas mapeia a URL para um destino
   │
   ▼
Middleware                               → autenticação, autorização, throttling
   │
   ▼
Camada de interface (Http/ ou Livewire/) → Controller / Componente Livewire
   │   FormRequest valida e normaliza a entrada
   ▼
Camada de aplicação (Actions / Use Cases)→ orquestra UM caso de uso
   │   recebe DTO, devolve DTO/entidade
   ▼
Domínio (Entities, Value Objects, Regras, Interfaces de Repositório)
   │
   ▼
Infraestrutura (Repositórios Eloquent, filas, e-mail, APIs externas)
   │
   ▼
Resposta (View Blade/Flux, Resource JSON ou redirect)
```

Regras do fluxo:

1. **Rota** não contém lógica; só aponta para um controller/componente.
2. **Controller/Componente** é fino: valida (FormRequest), chama **uma** Action e devolve a resposta.
3. **Action** contém o caso de uso. É a única que coordena domínio e infraestrutura.
4. **Domínio** guarda as regras de negócio puras (sem `Request`, `Auth`, `DB`, facades).
5. **Infraestrutura** implementa as interfaces definidas pelo domínio (ex.: `UserRepository` → `EloquentUserRepository`), ligadas no container de injeção de dependência.
6. Erros de negócio viram **exceções de domínio**, traduzidas para HTTP/UI no handler, nunca dentro da regra.

### Organização de pastas

```
app/
├── Domain/                      # Regras de negócio puras (sem framework)
│   └── <Contexto>/
│       ├── Entities/
│       ├── ValueObjects/
│       ├── Exceptions/
│       └── Repositories/        # Interfaces (contratos)
├── Application/                 # Casos de uso
│   └── <Contexto>/
│       ├── Actions/             # Ex.: CreateUserAction
│       └── DTOs/                # Dados de entrada/saída dos casos de uso
├── Infrastructure/              # Detalhes técnicos
│   ├── Persistence/
│   │   └── Eloquent/            # Models e implementações de repositório
│   └── Services/                # Integrações externas (e-mail, pagamento...)
├── Http/                        # Interface web/API
│   ├── Controllers/
│   ├── Requests/                # FormRequests (validação)
│   ├── Resources/               # Serialização JSON
│   └── Middleware/
├── Livewire/                    # Componentes de tela (interface)
└── Providers/                   # Bindings interface → implementação

resources/views/                 # Blade + Flux UI (somente apresentação)
database/                        # Migrations, factories, seeders
tests/{Unit,Feature}/            # Unit: Domain/Application · Feature: HTTP/Livewire
```

> Estrutura-alvo. Pastas são criadas conforme os contextos aparecem; não crie camadas vazias por antecipação.

**Regra de dependência:** `Http`/`Livewire` → `Application` → `Domain`. `Infrastructure` depende de `Domain` (implementa seus contratos). `Domain` não importa nada das outras camadas.

## Principais práticas

- **Uma responsabilidade por classe** e um caso de uso por Action.
- **Injeção de dependência** pelo construtor; dependa de interfaces, não de implementações.
- **Tipagem estrita**: `declare(strict_types=1);`, tipos em parâmetros e retornos.
- **Validação na borda** (FormRequest); o domínio garante suas próprias invariantes.
- **DTOs** para trafegar dados entre camadas, em vez de arrays soltos ou `Request`.
- **Eloquent fica na infraestrutura**; regras não vivem em Models nem em Controllers.
- **Transações** no nível da Action (`DB::transaction`), não no controller.
- **Sem lógica nas Views**: Blade/Flux apenas exibe; decisões ficam em componentes/Actions.
- **Segurança**: autorização por Policies/Gates, `$fillable` explícito, nunca confiar na entrada.
- **Testes**: regras de domínio e Actions com testes unitários; fluxos completos com testes de feature. Todo bug corrigido ganha um teste.
- **Migrations versionadas**: nunca altere uma migration já aplicada; crie outra.
- **Segredos só em `.env`**; nada sensível no repositório.
- **Formatação**: rodar `./vendor/bin/pint` antes de commitar.
- **Commits pequenos** com mensagem no imperativo (ex.: `Add create user action`).

## Guia de Clean Code

**Nomes**
- Revelam a intenção: `calculateOrderTotal()`, não `calc()` ou `doIt()`.
- Classes são substantivos (`InvoiceIssuer`); métodos são verbos (`issue()`); booleanos leem como pergunta (`isPaid`, `hasStock`).
- Sem abreviações obscuras nem sufixos vagos (`Manager`, `Helper`, `Util`).

**Funções e métodos**
- Pequenos e fazem **uma** coisa; um nível de abstração por função.
- Até ~3 parâmetros; acima disso, use um DTO.
- Sem *flag arguments* (`save($x, true)`): separe em dois métodos.
- Prefira **retorno antecipado** (guard clauses) a `if/else` aninhados.
- Sem efeitos colaterais escondidos: quem consulta não altera estado.

**Classes**
- Coesas e pequenas; se precisa da palavra "e" para descrevê-la, divida.
- Composição em vez de herança; herança só para relação "é um" real.
- Siga **SOLID**, principalmente SRP (uma razão para mudar) e DIP (dependa de abstrações).

**Código**
- Sem números/strings mágicos: use constantes, enums e Value Objects.
- **DRY** com bom senso: duplique até o padrão ficar claro, depois extraia.
- **YAGNI**: não implemente o que ainda não é necessário.
- Sem código morto ou comentado; o git guarda o histórico.
- Comentários explicam o **porquê**, não o **quê**; se precisa explicar o quê, renomeie ou extraia.
- Trate erros com **exceções** com nome de domínio; não retorne `null`/`false` para sinalizar falha.
- Evite aninhamento profundo e `else` desnecessário.

**Exemplo: controller fino + Action**

```php
// app/Http/Controllers/UserController.php
public function store(StoreUserRequest $request, CreateUserAction $action): RedirectResponse
{
    $action->execute(CreateUserData::fromRequest($request));

    return to_route('users.index');
}

// app/Application/Users/Actions/CreateUserAction.php
final class CreateUserAction
{
    public function __construct(private UserRepository $users) {}

    public function execute(CreateUserData $data): User
    {
        if ($this->users->existsByEmail($data->email)) {
            throw new EmailAlreadyInUse($data->email);
        }

        return $this->users->add(User::register($data->name, $data->email));
    }
}
```

## Checklist antes de abrir um commit

- [ ] A lógica está na camada certa (Action/Domínio, não no controller ou na view)?
- [ ] Nomes revelam a intenção e não há código morto?
- [ ] Há teste cobrindo o comportamento novo?
- [ ] `./vendor/bin/pint` e `php artisan test` passam?
