# docs/guides/ — X-Road style guide series

Mongolia X-Road instance `MN`-ийн **albani esnyi** гарын авлагын цуглуулга. NIIS-ийн `AR-*`, `IG-*`, `UG-*`, `UC-*` стандарт код-системээс санаа авч бичсэн.

## Гарын авлагын зам

```mermaid
flowchart TB
    AUDIENCE([Хэн та?]) --> ROLE{Үүрэг?}

    ROLE -->|Шинэ танилцагч| ENTRY[../taniltsuulga.md]
    ROLE -->|Архитектор| AR[ar-mn.md]
    ROLE -->|CS суулгагч| IG_CS[ig-cs.md]
    ROLE -->|SS суулгагч| IG_SS[ig-ss.md]
    ROLE -->|CS оператор| UG_CS[ug-cs.md]
    ROLE -->|SS оператор| UG_SS[ug-ss.md]
    ROLE -->|Use case-аар тулгарсан| UC[uc-mn.md]
    ROLE -->|Алдаа гарсан| TR[tr-mn.md]
    ROLE -->|Аудит / security review| SEC[sec-mn.md]

    classDef doc fill:#E3F2FD
    class ENTRY,AR,IG_CS,IG_SS,UG_CS,UG_SS,UC,TR,SEC doc
```

## Гарын авлагын файлын жагсаалт

| Код | Файл | Сэдэв | Үндсэн уншигч |
|---|---|---|---|
| AR-MN | [ar-mn.md](ar-mn.md) | Архитектур | Архитектор, SRE |
| IG-CS | [ig-cs.md](ig-cs.md) | CS суулгах | Системийн админ (one-time) |
| IG-SS | [ig-ss.md](ig-ss.md) | SS суулгах | Гишүүн org-ийн SRE |
| UG-CS | [ug-cs.md](ug-cs.md) | CS оператор UI | ҮДТ + Gerege ops |
| UG-SS | [ug-ss.md](ug-ss.md) | SS оператор UI | Гишүүн org оператор |
| UC-MN | [uc-mn.md](uc-mn.md) | Хэрэглээний хувилбарууд | Бүгд |
| TR-MN | [tr-mn.md](tr-mn.md) | Алдаа засах | Алдаатай үед бүгд |
| SEC-MN | [sec-mn.md](sec-mn.md) | Аюулгүй байдал | Security / compliance |

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
        AR_MN[ar-mn.md]
        IG_CS_MN[ig-cs.md]
        IG_SS_MN[ig-ss.md]
        UG_CS_MN[ug-cs.md]
        UG_SS_MN[ug-ss.md]
        UC_MN[uc-mn.md]
        TR_MN[tr-mn.md]
        SEC_MN[sec-mn.md]
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
