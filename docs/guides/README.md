# docs/guides/ — X-Road style guide series

Mongolia X-Road instance `MN`-ийн **albani esnyi** гарын авлагын цуглуулга. NIIS-ийн `AR-*`, `IG-*`, `UG-*`, `UC-*` стандарт код-системээс санаа авч бичсэн.

## Гарын авлагын зам

```mermaid
flowchart TB
    AUDIENCE([Хэн та?]) --> ROLE{Үүрэг?}

    ROLE -->|Шинэ танилцагч| ENTRY[../taniltsuulga.md]
    ROLE -->|Архитектор| AR[architecture.md]
    ROLE -->|CS суулгагч| IG_CS[install-central-server.md]
    ROLE -->|SS суулгагч| IG_SS[install-security-server.md]
    ROLE -->|CS оператор| UG_CS[operate-central-server.md]
    ROLE -->|SS оператор| UG_SS[operate-security-server.md]
    ROLE -->|Use case-аар тулгарсан| UC[use-cases.md]
    ROLE -->|Алдаа гарсан| TR[troubleshooting.md]
    ROLE -->|Аудит / security review| SEC[security.md]

    classDef doc fill:#E3F2FD
    class ENTRY,AR,IG_CS,IG_SS,UG_CS,UG_SS,UC,TR,SEC doc
```

## Гарын авлагын файлын жагсаалт

| Код | Файл | Сэдэв | Үндсэн уншигч |
|---|---|---|---|
| AR-MN | [architecture.md](architecture.md) | Архитектур | Архитектор, SRE |
| IG-CS | [install-central-server.md](install-central-server.md) | CS суулгах | Системийн админ (one-time) |
| IG-SS | [install-security-server.md](install-security-server.md) | SS суулгах | Гишүүн org-ийн SRE |
| UG-CS | [operate-central-server.md](operate-central-server.md) | CS оператор UI | ҮДТ + Gerege ops |
| UG-SS | [operate-security-server.md](operate-security-server.md) | SS оператор UI | Гишүүн org оператор |
| UC-MN | [use-cases.md](use-cases.md) | Хэрэглээний хувилбарууд | Бүгд |
| TR-MN | [troubleshooting.md](troubleshooting.md) | Алдаа засах | Алдаатай үед бүгд |
| SEC-MN | [security.md](security.md) | Аюулгүй байдал | Security / compliance |

## NIIS-ийн оригинал docs-той харьцуулалт

```mermaid
graph LR
    subgraph niis["NIIS upstream (xroad-public/docs)"]
        AR_CS_NIIS[AR-CS]
        AR_SS_NIIS[AR-SS]
        IG_CS_NIIS[IG-CS]
        IG_SS_NIIS[IG-SS]
        UG_CS_NIIS[UG-CS]
        UG_SS_NIIS[UG-SS]
        UC_NIIS[UC-MEMBER, UC-SS, UC-FED]
        PR_MESS[PR-MESS]
        PR_GCONF[PR-GCONF]
        DM_CS[DM-CS]
        DM_SS[DM-SS]
    end

    subgraph mn["Mongolia MN guides (this folder)"]
        AR_MN[architecture.md]
        IG_CS_MN[install-central-server.md]
        IG_SS_MN[install-security-server.md]
        UG_CS_MN[operate-central-server.md]
        UG_SS_MN[operate-security-server.md]
        UC_MN[use-cases.md]
        TR_MN[troubleshooting.md]
        SEC_MN[security.md]
    end

    AR_CS_NIIS --> AR_MN
    AR_SS_NIIS --> AR_MN
    IG_CS_NIIS --> IG_CS_MN
    IG_SS_NIIS --> IG_SS_MN
    UG_CS_NIIS --> UG_CS_MN
    UG_SS_NIIS --> UG_SS_MN
    UC_NIIS --> UC_MN

    style PR_MESS stroke-dasharray: 5 5
    style PR_GCONF stroke-dasharray: 5 5
    style DM_CS stroke-dasharray: 5 5
    style DM_SS stroke-dasharray: 5 5
```

Dashed-аар тэмдэглэсэн (PR-*, DM-*) нь NIIS upstream-ийн protocol/data-model docs-ыг шууд reference хийж, Mongolia-ийн контекстэд тусдаа доку хэлбэрээр давтаагүй. Хэрэг гарвал: https://github.com/nordic-institute/X-Road/tree/develop/doc.

## Авч үлдэх гарын авлага бэлдэх ажил (хувийн TODO)

- [ ] PR-GCONF-MN — globalconf protocol Mongolia-specific tweaks
- [ ] DM-CS-MN — centerui DB schema Mongolia-side
- [ ] DM-SS-MN — serverconf DB schema
- [ ] OPMON-MN — Operational Monitoring (Phase-3)
- [ ] FED-MN — Federation onboarding (Phase-3)

---

*Гарын авлага бүр нэг шинэлэг сэдвийг гүн авч үздэг. Шинэ гарын авлага бэлдэх хэрэгцээ гарвал энэхүү index-д нэмж бичнэ.*
