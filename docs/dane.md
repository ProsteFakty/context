# Jakie dane i gdzie je przechowujemy

Backend jest taki sam jak w `proste-fakty`, więc dane i bazy też są te same.

## Co przechowujemy

- **Dane z Sejmu** — posiedzenia, projekty ustaw i wyniki głosowań. Trzymamy ich kopię u siebie, bo API Sejmu jest wolne: pobieramy raz, potem aplikacja czyta już z naszej bazy.
- **Ustawy** — pocięte na artykuły: oryginalny tekst, streszczenie prostym językiem i odesłania do innych artykułów.
- **Wektory streszczeń artykułów** — do wyszukiwarki.
- **Teksty wygenerowane przez Gemini** (streszczenia, opisy posiedzeń) — zapisujemy je, żeby nie generować i nie płacić drugi raz.

Nie przechowujemy żadnych danych użytkowników (brak kont).

## Gdzie

| Magazyn | Co trzyma |
|---|---|
| **PostgreSQL** | Dane z Sejmu, ustawy i artykuły, teksty od Gemini |
| **Qdrant** | Wektory do wyszukiwania |
| **S3** | Backupy baz i pliki źródłowe ustaw (PDF) |
| **Telefon** | Nic — aplikacja tylko czyta z API |

## RAG (z `proste-fakty`)

```mermaid
flowchart LR
    U["Użytkownik"] -->|pytanie| API["FastAPI"]
    API -->|wyszukiwanie semantyczne| VS["Qdrant"]
    VS -->|super-chunki + kontekst| LLM["Gemini"]
    LLM -->|dociągnięcie brakującego artykułu| PG["PostgreSQL"]
    PG --> LLM
    LLM -->|odpowiedź po ludzku| U
```

- **Szybka ścieżka — Qdrant.** Super-chunki: główny artykuł wzbogacony o kontekst artykułów powiązanych. W ok. 90% przypadków daje LLM-owi cały potrzebny kontekst.
- **Ścieżka awaryjna — PostgreSQL.** Surowe, pocięte artykuły z unikalnym identyfikatorem (slug). Gdy w super-chunku jest odesłanie do innej ustawy, model dociąga brakujący artykuł przez Function Calling.

## Na AWS (tanio)

Zasada: **jedna maszyna EC2 z tym samym `docker compose` co lokalnie** (Postgres + Qdrant + backend + Caddy), bez usług zarządzanych. Danych jest mało (seed: ~10 tys. wierszy, snapshot Qdranta ~0,5 MB), ruch mały, więc RDS, Qdrant Cloud czy load balancer to koszt bez zysku.

```mermaid
flowchart LR
    APP["Aplikacja mobilna"] -->|HTTPS 443| CADDY

    subgraph EC2["EC2 t4g.medium — eu-central-1 (Frankfurt)"]
        CADDY["Caddy<br/>HTTPS Let's Encrypt"] --> BE["backend<br/>FastAPI + bge-m3"]
        BE --> PG[("postgres")]
        BE --> QD[("qdrant")]
    end

    DNS[["Domena u dowolnego rejestratora<br/>rekord A → Elastic IP"]] -.-> CADDY
    BE -->|streszczenia Sejmu| GEM(["Gemini API"])
    BE -->|dane Sejmu| SEJM[["api.sejm.gov.pl"]]
    EC2 -->|cron: pg_dump + snapshot Qdranta<br/>rola IAM| S3[("S3<br/>backups/, pdfs/")]

    DEV["Laptop dewelopera<br/>pipeline ingestu"] -->|seed.sql + snapshot| EC2
    DEV --> S3
```

### Elementy

| Element | Wybór | Dlaczego |
|---|---|---|
| Maszyna | **EC2 `t4g.medium`** (2 vCPU Graviton/ARM, 4 GB RAM), Ubuntu 24.04 arm64 | Najtańsza z 4 GB RAM. ARM jest tańszy od x86, a obrazy `postgres`, `qdrant` i Python/torch mają wersje arm64 |
| Tryb kredytów CPU | **`standard`** (nie `unlimited`) | `t4g` domyślnie startuje w `unlimited`: przy długim obciążeniu CPU AWS dolicza opłaty. W `standard` maszyna co najwyżej zwolni, a rachunek się nie zmieni |
| Region | `eu-central-1` (Frankfurt) | Najbliżej użytkowników w Polsce |
| Dysk | **EBS gp3, 30 GB** | System + Docker + model (~2 GB) + bazy (MB) + swap 2 GB, z zapasem |
| Adres | **Elastic IP** | Stały adres pod rekord A domeny; nie zmienia się po restarcie maszyny |
| Firewall | Security group: 80, 443 z całego świata, 22 tylko z naszych IP | Postgres i Qdrant niewidoczne z zewnątrz |
| Postgres | Kontener `postgres:17` | RDS `db.t4g.micro` to dodatkowy rachunek co miesiąc przy kilku MB danych |
| Qdrant | Kontener `qdrant/qdrant:v1.19.1` | Zero kosztu, ta sama wersja co snapshot |
| HTTPS | Caddy w compose, certyfikat Let's Encrypt | Darmowe. Application Load Balancer kosztowałby prawie tyle co sama maszyna |
| Domena / DNS | Dowolny rejestrator, rekord A → Elastic IP | Route 53 niepotrzebny |
| Backup | Cron: `db/dump.sh` → `aws s3 cp` do S3; dostęp przez **rolę IAM** maszyny | Kilka MB dziennie, grosze. Bez kluczy AWS na dysku. Snapshoty EBS niepotrzebne — dane i tak odtwarzamy z dumpa |
| Ingest | **Na laptopie**, nie na serwerze | Ciężką pracę (Gemini + embeddingi całych ustaw) robimy lokalnie. Na serwer trafia gotowy `seed.sql` + snapshot (`db/restore.sh --postgres`) |

### Koszt miesięczny (szacunek)

| Pozycja | Koszt |
|---|---:|
| EC2 `t4g.medium`, on-demand (~730 h) | ≈ $28 |
| EBS gp3 30 GB | ≈ $3 |
| Publiczny IPv4 (Elastic IP) | ≈ $3,6 |
| S3: backupy + PDF-y (< 1 GB) | < $0,10 |
| Transfer wychodzący (100 GB/mies. gratis) | $0 |
| Certyfikat, DNS u rejestratora | $0 |
| **Razem AWS** | **≈ $35 / mies.** |

Taniej: **Savings Plan na 1 rok bez przedpłaty** obniża cenę maszyny o ok. 30–40%, czyli razem wychodzi ok. $25–28. Warto go kupić dopiero, gdy wiemy, że 4 GB wystarczą.

Poza AWS: Gemini API (tylko opisy posiedzeń i streszczenia projektów, każde generowane raz) i domena. Cena `t4g.medium` we Frankfurcie to szacunek przeliczony ze stawki `t4g.xlarge` (źródło: [devzero.io](https://www.devzero.io/instances/aws/t4g.large), stan 10.2026). Przed startem sprawdzić w [AWS Pricing Calculator](https://calculator.aws/).

### Na co uważać

- **RAM.** Sam model bge-m3 to ~2,3 GB, do tego torch, Postgres i Qdrant. 4 GB powinno wystarczyć (nie sprawdzaliśmy tego pomiarem), ale dodajemy **2 GB swapu**. Jeśli backend będzie padał przez brak pamięci, zmieniamy typ na `t4g.large` (8 GB, ≈ $56 za maszynę). Zmiana typu to zatrzymanie i start maszyny, dane na EBS zostają.
- **Porty.** W `docker-compose.yml` z `proste-fakty` Postgres (5432) i Qdrant (6333/6334) są wystawione na zewnątrz. Na serwerze **usuwamy `ports` z tych usług**. Security group to druga linia obrony, nie jedyna.
- **Hasła.** Zmieniamy domyślne `postgres/postgres`, `GEMINI_API_KEY` trzymamy w `.env` poza repo.
- **Elastic IP** jest płatny także wtedy, gdy maszyna stoi wyłączona. Gdy rezygnujemy z serwera, zwalniamy adres.
- **Pierwszy start.** Model pobiera się raz do volume `hf_cache` (~2 GB), a po restarcie backendu ładuje się ~10–40 s.
- **Alarm budżetu.** W AWS Budgets ustawiamy alert np. na $40/mies. Jest darmowy i od razu pokaże, jeśli coś zacznie kosztować więcej, niż zakładamy.
