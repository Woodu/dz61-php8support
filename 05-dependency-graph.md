# Dependency Graph - Feature Migration Order

**Purpose:** Visual representation of feature dependencies
**Format:** Mermaid diagrams
**Last Updated:** 2026-02-13

---

## Overview

This document shows the dependency relationships between all features. Features must be migrated in dependency order - a feature can only be started when all its dependencies are complete.

**Legend:**
- 🔴 P0 - Critical (must migrate first)
- 🟡 P1 - Core (migrate after P0)
- 🟢 P2 - Important (migrate after P1)
- 🔵 P3 - Enhanced (migrate after P2)

---

## High-Level Overview

```mermaid
graph TD
    A[🔴 P0: Infrastructure] --> B[🟡 P1: Core Forum]
    B --> C[🟢 P2: Moderation/Admin]
    C --> D[🔵 P3: Enhanced Features]

    A --> A1[Configuration]
    A --> A2[Bootstrap]
    A --> A3[Database Layer]
    A2 --> A4[Authentication]
    A2 --> A5[Caching]
    A2 --> A6[Session Management]
    A4 --> A7[User Login]

    B --> B1[Forum Navigation]
    B --> B2[Thread Management]
    B --> B3[Posting System]
    B --> B4[User Management]
    B1 --> B2
    B2 --> B3

    C --> C1[Moderation System]
    C --> C2[Search System]
    C --> C3[Private Messaging]
    C --> C4[Attachment System]
    C --> C5[Admin Control Panel]
    C1 --> C5

    D --> D1[Social Features]
    D --> D2[Gamification]
    D --> D3[Template System]
    D --> D4[Plugins]
```

---

## P0 Critical Dependencies

```mermaid
graph TD
    A1[Configuration] --> A2[Core Bootstrap]
    A2 --> A3[Database Layer]
    A2 --> A4[Caching System]
    A2 --> A5[Session Management]

    A3 --> A6[User Login]
    A4 --> A6
    A5 --> A6

    A6 --> A7[Authentication Complete]

    style A1 fill:#ff6b6b
    style A2 fill:#ff6b6b
    style A3 fill:#ff6b6b
    style A4 fill:#ff6b6b
    style A5 fill:#ff6b6b
    style A6 fill:#ff6b6b
    style A7 fill:#51cf66
```

---

## P1 Core Dependencies

```mermaid
graph TD
    P0[P0 Complete] --> B1[Forum Index]
    P0 --> B2[Member Profile]
    P0 --> B3[Online Users]

    B1 --> B4[Forum Display]
    B4 --> B5[Thread Viewing]

    B5 --> B6[New Thread]
    B5 --> B7[New Reply]
    B5 --> B8[Edit Post]

    B3 --> B9[Post Controller]
    B6 --> B9
    B7 --> B9
    B8 --> B9

    B9 --> B10[Post Functions]
    B10 --> B11[Attachment Upload]
    B10 --> B12[DiscuzCode Parser]

    B2 --> B13[User CP]
    B13 --> B14[User Registration]
    B13 --> B15[Avatar System]

    style P0 fill:#ff6b6b
    style B1 fill:#ffd93d
    style B2 fill:#ffd93d
    style B3 fill:#ffd93d
    style B4 fill:#ffd93d
    style B5 fill:#ffd93d
    style B6 fill:#ffd93d
    style B7 fill:#ffd93d
    style B8 fill:#ffd93d
    style B9 fill:#ffd93d
    style B10 fill:#ffd93d
    style B11 fill:#ffd93d
    style B12 fill:#ffd93d
    style B13 fill:#ffd93d
    style B14 fill:#ffd93d
    style B15 fill:#ffd93d
```

---

## P2 Moderation Dependencies

```mermaid
graph TD
    P1[P1 Complete] --> C1[ModCP Index]
    P0 --> C1

    C1 --> C2[ModCP Login]
    C1 --> C3[Forum Access Control]
    C1 --> C4[Thread Moderation]

    C4 --> C5[Post Moderation]
    C5 --> C6[Edit Post Mod]

    C1 --> C7[Member Banning]
    C1 --> C8[Report System]
    C1 --> C9[ModCP Logs]
    C1 --> C10[Prune Threads]

    style P0 fill:#ff6b6b
    style P1 fill:#ffd93d
    style C1 fill:#6BCB77
    style C2 fill:#6BCB77
    style C3 fill:#6BCB77
    style C4 fill:#6BCB77
    style C5 fill:#6BCB77
    style C6 fill:#6BCB77
    style C7 fill:#6BCB77
    style C8 fill:#6BCB77
    style C9 fill:#6BCB77
    style C10 fill:#6BCB77
```

---

## P2 Search Dependencies

```mermaid
graph TD
    P1[P1 Complete] --> S1[Search Controller]
    B4[Forum Display] --> S1
    B5[Thread Viewing] --> S1

    S1 --> S2[Fulltext Search]
    S1 --> S3[Search Cache]

    A4[Caching System] --> S3

    style P1 fill:#ffd93d
    style B4 fill:#ffd93d
    style B5 fill:#ffd93d
    style A4 fill:#ff6b6b
    style S1 fill:#6BCB77
    style S2 fill:#6BCB77
    style S3 fill:#6BCB77
```

---

## P2 PM & Attachments Dependencies

```mermaid
graph TD
    B2[Member Profile] --> PM1[PM System]
    P1[P1 Complete] --> PM1

    PM1 --> PM2[PM Sending]
    PM1 --> PM3[PM Folder Management]
    PM1 --> PM4[PM Blacklist]

    B9[Post Controller] --> ATT1[Attachment Handler]
    B10[Post Functions] --> ATT1

    ATT1 --> ATT2[Thumbnail Generation]
    ATT2 --> ATT3[Image Processing]

    style B2 fill:#ffd93d
    style P1 fill:#ffd93d
    style B9 fill:#ffd93d
    style B10 fill:#ffd93d
    style PM1 fill:#6BCB77
    style PM2 fill:#6BCB77
    style PM3 fill:#6BCB77
    style PM4 fill:#6BCB77
    style ATT1 fill:#6BCB77
    style ATT2 fill:#6BCB77
    style ATT3 fill:#6BCB77
```

---

## P2 AdminCP Dependencies

```mermaid
graph TD
    P0[P0 Complete] --> AD1[AdminCP Login]
    AD1 --> AD2[AdminCP Index]

    AD2 --> AD3[Forum Management]
    AD2 --> AD4[User Group Management]
    AD2 --> AD5[Member Management]
    AD2 --> AD6[Database Management]
    AD2 --> AD7[Settings Management]
    AD2 --> AD8[Template Management]
    AD2 --> AD9[Plugin Management]
    AD2 --> AD10[Announcement Management]
    AD2 --> AD11[Thread Management]
    AD2 --> AD12[Attachment Management]
    AD2 --> AD13[Moderation Queue]
    AD2 --> AD14[Statistics]
    AD2 --> AD15[Prune Tools]
    AD2 --> AD16[Logs Management]
    AD2 --> AD17[Smilies Management]
    AD2 --> AD18[FAQ Management]
    AD2 --> AD19[Advertisement Management]
    AD2 --> AD20[Credit System]
    AD2 --> AD21[Medals Management]
    AD2 --> AD22[Magic Items]
    AD2 --> AD23[JS/CSS Wizard]
    AD2 --> AD24[Run Wizard]
    AD2 --> AD25[Tools]
    AD2 --> AD26[Project/Update]
    AD2 --> AD27[Quick Queries]
    AD2 --> AD28[Recycle Bin]
    AD2 --> AD29[Check Tools]
    AD2 --> AD30[Thread Types]

    style P0 fill:#ff6b6b
    style AD1 fill:#6BCB77
    style AD2 fill:#6BCB77
    style AD3 fill:#6BCB77
    style AD4 fill:#6BCB77
    style AD5 fill:#6BCB77
    style AD6 fill:#6BCB77
    style AD7 fill:#6BCB77
    style AD8 fill:#6BCB77
    style AD9 fill:#6BCB77
    style AD10 fill:#6BCB77
    style AD11 fill:#6BCB77
    style AD12 fill:#6BCB77
    style AD13 fill:#6BCB77
    style AD14 fill:#6BCB77
    style AD15 fill:#6BCB77
    style AD16 fill:#6BCB77
    style AD17 fill:#6BCB77
    style AD18 fill:#6BCB77
    style AD19 fill:#6BCB77
    style AD20 fill:#6BCB77
    style AD21 fill:#6BCB77
    style AD22 fill:#6BCB77
    style AD23 fill:#6BCB77
    style AD24 fill:#6BCB77
    style AD25 fill:#6BCB77
    style AD26 fill:#6BCB77
    style AD27 fill:#6BCB77
    style AD28 fill:#6BCB77
    style AD29 fill:#6BCB77
    style AD30 fill:#6BCB77
```

---

## P3 Social Features Dependencies

```mermaid
graph TD
    B2[Member Profile] --> SOC1[User Space]
    SOC1 --> SOC2[Buddy List]
    B4[Forum Display] --> SOC3[User Statistics]
    SOC3 --> SOC4[Rank List]
    B4[Forum Display] --> SOC5[Digest Display]
    B6[New Thread] --> SOC6[Tag System]

    style B2 fill:#ffd93d
    style B4 fill:#ffd93d
    style B6 fill:#ffd93d
    style SOC1 fill:#4d96ff
    style SOC2 fill:#4d96ff
    style SOC3 fill:#4d96ff
    style SOC4 fill:#4d96ff
    style SOC5 fill:#4d96ff
    style SOC6 fill:#4d96ff
```

---

## P3 Gamification Dependencies

```mermaid
graph TD
    B2[Member Profile] --> GM1[User Management Base]
    GM1 --> GM2[Credit System]

    GM2 --> GM3[Bank System]
    GM2 --> GM4[Reward System]
    GM1 --> GM5[Medal System]
    GM1 --> GM6[Magic Items]

    style B2 fill:#ffd93d
    style GM1 fill:#4d96ff
    style GM2 fill:#4d96ff
    style GM3 fill:#4d96ff
    style GM4 fill:#4d96ff
    style GM5 fill:#4d96ff
    style GM6 fill:#4d96ff
```

---

## P3 Template & Plugin Dependencies

```mermaid
graph TD
    A4[Caching System] --> TMP1[Template Engine]
    TMP1 --> TMP2[Chinese Language]
    TMP1 --> TMP3[Frame Support]
    TMP2 --> TMP4[Multi-byte Support]

    A2[Core Bootstrap] --> PLG1[Plugin Handler]
    PLG1 --> PLG2[Bank Plugin]
    PLG1 --> PLG3[Sitemap Plugin]
    PLG1 --> PLG4[Admin Plugins]
    PLG1 --> PLG5[MOC Plugin]

    style A4 fill:#ff6b6b
    style A2 fill:#ff6b6b
    style TMP1 fill:#4d96ff
    style TMP2 fill:#4d96ff
    style TMP3 fill:#4d96ff
    style TMP4 fill:#4d96ff
    style PLG1 fill:#4d96ff
    style PLG2 fill:#4d96ff
    style PLG3 fill:#4d96ff
    style PLG4 fill:#4d96ff
    style PLG5 fill:#4d96ff
```

---

## P3 Custom PoketTB Features Dependencies

```mermaid
graph TD
    B2[Member Profile] --> CT1[User Base]
    GM2[Credit System] --> CT2[Pet System]
    GM2 --> CT3[DEX System]
    CT1 --> CT4[Helper System]
    CT1 --> CT5[Campaign System]
    CT1 --> CT6[Family System]
    ATT1[Attachment Handler] --> CT7[Event Download]

    style B2 fill:#ffd93d
    style GM2 fill:#4d96ff
    style ATT1 fill:#6BCB77
    style CT1 fill:#4d96ff
    style CT2 fill:#4d96ff
    style CT3 fill:#4d96ff
    style CT4 fill:#4d96ff
    style CT5 fill:#4d96ff
    style CT6 fill:#4d96ff
    style CT7 fill:#4d96ff
```

---

## P3 UC Integration Dependencies

```mermaid
graph TD
    A6[User Login] --> UC1[UC Client]
    PM1[PM System] --> UC2[PM Sync]
    B15[Avatar System] --> UC3[Avatar Sync]
    B14[User Registration] --> UC4[Member Sync]

    UC1 --> UC2
    UC1 --> UC3
    UC1 --> UC4

    style A6 fill:#ff6b6b
    style PM1 fill:#6BCB77
    style B15 fill:#ffd93d
    style B14 fill:#ffd93d
    style UC1 fill:#4d96ff
    style UC2 fill:#4d96ff
    style UC3 fill:#4d96ff
    style UC4 fill:#4d96ff
```

---

## Critical Path Analysis

### Absolute Critical Path (Must Follow)

```mermaid
graph LR
    A[Configuration] --> B[Bootstrap]
    B --> C[Database]
    B --> D[Caching]
    B --> E[Session]
    C --> F[Login]
    D --> F
    E --> F
    F --> G[Auth Complete]

    G --> H[Forum Index]
    H --> I[Forum Display]
    I --> J[Thread View]
    J --> K[Post Controller]

    style A fill:#ff6b6b
    style B fill:#ff6b6b
    style C fill:#ff6b6b
    style D fill:#ff6b6b
    style E fill:#ff6b6b
    style F fill:#ff6b6b
    style G fill:#51cf66
    style H fill:#ffd93d
    style I fill:#ffd93d
    style J fill:#ffd93d
    style K fill:#ffd93d
```

**Timeline:** Days 1-40

---

## Parallel Development Opportunities

After Day 40 (P1 core complete), these can be developed in parallel:

### Team A (Moderation)
- ModCP System
- Search System
- Report System

### Team B (Admin)
- AdminCP
- Settings Management
- User Management

### Team C (User Features)
- Private Messaging
- Attachments
- User Profile Enhancements

---

## Dependency Risk Assessment

### High Risk (Single Point of Failure)
1. **Core Bootstrap** - Everything depends on this
2. **Database Layer** - All data access
3. **Authentication** - All protected features

### Medium Risk (Multiple Dependencies)
1. **Forum Display** - 15+ features depend
2. **Post Controller** - 10+ features depend
3. **User Management** - 20+ features depend

### Low Risk (Few Dependencies)
1. Most P3 features
2. Standalone plugins
3. Template customizations

---

## Migration Sequence Summary

### Phase 1: Foundation (Days 1-15)
- P0 Infrastructure
- No parallel work possible

### Phase 2: Core (Days 16-50)
- P1 Features
- Limited parallel work after Day 30

### Phase 3: Important (Days 51-80)
- P2 Features
- Heavy parallel work possible

### Phase 4: Enhanced (Days 81-120)
- P3 Features
- Maximum parallel work

---

## Notes

1. **Circular Dependencies:** None detected
2. **Orphan Features:** None (all have dependencies)
3. **Missing Dependencies:** None detected
4. **Recommendation:** Follow critical path strictly for first 40 days

---

## Dependency Matrix

| Feature | Depends On | Blocks | Risk Level |
|---------|-----------|--------|-----------|
| Configuration | None | Bootstrap, DB | 🔴 High |
| Bootstrap | Configuration | All features | 🔴 High |
| Database Layer | Bootstrap | All data features | 🔴 High |
| Authentication | Bootstrap, DB, Cache, Session | All protected features | 🔴 High |
| Forum Index | P0 | All forum features | 🟡 Medium |
| Forum Display | Forum Index | Thread view, post, mod | 🟡 Medium |
| Thread Viewing | Forum Display | Posting, moderation | 🟡 Medium |
| Post Controller | Thread view, New thread | Attachments, moderation | 🟡 Medium |
| User Management | P0 | All user features | 🟡 Medium |
| AdminCP | P0 | All admin features | 🟢 Low |
| Plugins | P0 | All plugins | 🟢 Low |
| Social Features | P1, User Mgmt | None | 🟢 Low |
| Gamification | P1, User Mgmt | Custom features | 🟢 Low |
| Template System | P0, Cache | Theme customization | 🟢 Low |
| UC Integration | P0, P1 features | Sync features | 🟢 Low |

---

**Document Status:** ✅ Complete
**Next Document:** 06-execution-sprints.md
