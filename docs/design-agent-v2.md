# Cozystack User Apps — UI Design Specification v2

Ты дизайнер интерфейсов. Создай HTML-мок для Cozystack User Apps —
self-hosted PaaS для провайдеров (ISP).

Целевой пользователь: клиент провайдера, нетехнический. Он не знает что такое
Kubernetes, namespace, YAML. Хочет просто запустить WordPress или Node.js-сайт
как в Heroku/Vercel — выбрал, настроил, нажал кнопку.

---

## СЦЕНАРИИ (8 экранов, один long-scroll HTML)

1. **My Applications** — список запущенных приложений с их статусами
2. **Application Details** — детали приложения, статус, values, actions
3. **Action Modal** — модалка выполнения action с формой параметров
4. **App Store** — каталог шаблонов для запуска нового
5. **Template Page** — описание шаблона, требования, кнопка "Launch"
6. **Launch Form** — форма запуска приложения
7. **Environments** — список окружений с их статусами
8. **Create Environment** — форма создания нового окружения

Между экранами — визуальный разделитель с подписью экрана (для презентации).

---

## ДИЗАЙН-СИСТЕМА

### Цвета

```css
--bg-app:         #1a1a1e;
--bg-sidebar:     #16161a;
--bg-surface:     #222226;
--bg-elevated:    #2a2a2e;
--bg-hover:       #2e2e34;
--text-primary:   #e8e8ea;
--text-secondary: #9999a8;
--text-muted:     #666672;
--border:         rgba(255,255,255,0.08);
--border-hover:   rgba(255,255,255,0.14);
--accent-blue:    #4c7ef3;
--accent-green:   #3ecf8e;
--accent-amber:   #f5a623;
--accent-red:     #f87171;
--accent-purple:  #a78bfa;
```

### Типографика

- Шрифт: system-ui / -apple-system
- Логотип навбара: "COZY≡TACK", 14px, font-weight 700, letter-spacing 0.12em, uppercase
- H1 страниц: 22px, font-weight 600, color text-primary
- H2 секций: 16px, font-weight 600
- Лейблы полей: 11px, font-weight 500, color text-muted, letter-spacing 0.06em
- Body: 13px, font-weight 400
- Мелкий/вспомогательный: 12px, color text-secondary
- Версии, slug-и: font-family monospace, 11-12px

### Компоненты

**Верхняя панель** (height 48px, bg-sidebar, border-bottom):

- Слева: логотип COZY≡TACK
- Правее: dropdown окружения "My Space" (не "default" — для нетехника)
- Справа: иконки уведомлений + аватар "kvaps"

**Sidebar** (176px, bg-sidebar):

- Без иконок
- Секции: Applications / Environments / Resources / Backups / Settings
- "Applications" раскрыта: My Apps (активный), App Store
- Активный пункт: bg-elevated, font-weight 500
- Hover: bg-hover

**Status badge** (height 22px, padding 0 8px, border-radius 5px, font-size 11px):

- Running:      bg rgba(62,207,142,0.15),  color #3ecf8e
- Installing:   bg rgba(76,126,243,0.15),  color #4c7ef3
- Upgrading:    bg rgba(76,126,243,0.15),  color #4c7ef3
- Starting:     bg rgba(245,166,35,0.15),  color #f5a623
- Stopped:      bg rgba(255,255,255,0.06), color #9999a8
- Failed:       bg rgba(248,113,113,0.15), color #f87171
- Removing:     bg rgba(248,113,113,0.15), color #f87171

**Environment status badge**:

- Ready:        bg rgba(62,207,142,0.15),  color #3ecf8e
- Provisioning: bg rgba(76,126,243,0.15),  color #4c7ef3
- Degraded:     bg rgba(245,166,35,0.15),  color #f5a623
- Deleting:     bg rgba(248,113,113,0.15), color #f87171

**Category badge** (height 20px, padding 0 7px, border-radius 4px, font-size 11px):

- cms:     bg rgba(52,130,246,0.15),  color #6aaaf8
- nodejs:  bg rgba(62,207,142,0.15),  color #5dd6a0
- php:     bg rgba(144,97,249,0.15),  color #b08af9
- static:  bg rgba(245,166,35,0.15),  color #f5c462
- game:    bg rgba(248,113,113,0.15), color #f87171

**App-карточка в "My Apps"** (bg-surface, border 0.5px, border-radius 10px, padding 16px):

- Строка 1: иконка (32px) + название приложения + status badge справа
- Строка 2: тип шаблона (12px, text-muted) + домен (12px, accent-blue, как ссылка)
- Строка 3: "Launched 3 days ago" + Environment badge + кнопки [Open ↗] [Manage] [⋮]
- Hover: border-color → border-hover
- Клик на карточку → Application Details

**Environment-карточка** (bg-surface, border 0.5px, border-radius 10px, padding 16px):

- Строка 1: название + status badge справа
- Строка 2: ресурсы (nodes, CPU, RAM) в виде pills
- Строка 3: "Created 5 days ago" + кнопки [Manage] [⋮]
- Hover: border-color → border-hover

**Каталог-карточка в "App Store"** (bg-surface, border 0.5px, border-radius 10px, padding 16px):

- Иконка: 44×44px, border-radius 10px, цветной bg opacity 0.12
- Название: 14px, font-weight 500
- Category badge под названием
- Описание: 12px, text-secondary, 2 строки max
- Кнопка [Launch →] в правом нижнем углу

**Форма** (без YAML, без технических терминов):

- Input: height 34px, bg bg-elevated, border 0.5px border-hover,
  border-radius 6px, padding 0 10px, font-size 13px
- Select: то же + стрелка вниз справа (SVG, без appearance)
- Toggle: 32×18px, border-radius 9px; on → #4c7ef3, off → bg-hover,
  ползунок white 14×14px
- Password input: type=password, иконка глаза справа
- Лейбл 🔒 рядом с чувствительными полями (opacity 0.35)
- Hint под полем: 11px, text-muted

**Кнопки**:

- Primary: bg #4c7ef3, color white, height 32px, padding 0 14px,
  border-radius 6px, font-size 13px, font-weight 500
- Default: bg transparent, border 0.5px border-hover, color text-primary
- Danger: border rgba(248,113,113,0.3), color #f87171
- Нет теней, нет градиентов

**Модалка**:

- Overlay: bg rgba(0,0,0,0.6)
- Content: bg-surface, border-radius 12px, max-width 480px
- Header: padding 20px 24px, border-bottom, title 16px font-weight 600
- Body: padding 24px
- Footer: padding 16px 24px, border-top, кнопки справа

**Разделители**: 0.5px solid rgba(255,255,255,0.08) везде

---

## КОНТЕНТ ПО ЭКРАНАМ

### Экран 1 — My Applications

Хлебные крошки: Applications
Заголовок: "My Applications"
Кнопка [+ Launch New App] справа от заголовка

4 карточки запущенных приложений:

1. 🌐 my-blog          | WordPress   | Running    | my-blog.example.com
   "WordPress · production · Launched 5 days ago"    [Open ↗] [Manage] [⋮]

2. 🛒 shop             | WooCommerce | Installing | shop.example.com
   "WooCommerce · production · Launched 2 minutes ago"  [Open ↗] [Manage] [⋮]

3. 🎮 minecraft-server | Minecraft   | Running    | —
   "Minecraft · staging · Launched 1 day ago"  [Connect] [Manage] [⋮]

4. 📄 landing          | Static Site | Stopped    | landing.example.com
   "Static Site · production · Launched 12 days ago"  [Start] [Manage] [⋮]

Под карточками: мелкий текст "4 applications · 2 running"

---

### Экран 2 — Application Details

Хлебные крошки: Applications > my-blog
Заголовок: "my-blog" + status badge [Running]
Кнопка [Actions ▾] справа (dropdown)

**Секция Status** (карточка):

| Field         | Value                           |
|---------------|---------------------------------|
| Status        | Running                         |
| Environment   | production                      |
| Template      | WordPress                       |
| Domain        | my-blog.example.com (link)      |
| Created       | January 10, 2024 at 14:32       |
| Last updated  | January 12, 2024 at 09:15       |

**Секция Configuration** (карточка):

Заголовок: "Configuration" + кнопка [Edit] справа

| Parameter       | Value                    |
|-----------------|--------------------------|
| Site title      | My Awesome Blog          |
| Admin email     | admin@example.com        |
| Storage size    | 10 GB                    |
| Enable cache    | Yes                      |

**Actions dropdown содержит**:

- 🔄 Restart
- 📦 Create Backup
- 🗑️ Delete (danger)

---

### Экран 3 — Action Modal (Backup)

Модалка поверх Application Details.

Header: "Create Backup"
Subtitle: "Create a backup of your WordPress site"

Форма:

- Backup name*     placeholder "before-update"
  hint: "A short name to identify this backup"

Footer: [Cancel] [Create Backup]

---

### Экран 4 — App Store

Хлебные крошки: Applications > App Store
Заголовок: "App Store"
Строка поиска + фильтры: All / CMS / Node.js / PHP / Static / Games

Сетка карточек (3 колонки, 8 штук):

1. 🔵 WordPress    [cms]         "The world's most popular CMS"
2. 🟣 Drupal       [cms][php]    "Enterprise content management"
3. 👻 Ghost        [cms]         "Modern publishing platform"
4. 🟢 Node.js App  [nodejs]      "Deploy any Node.js application"
5. ⚡ Next.js      [nodejs]      "Full-stack React framework"
6. 📄 Static Site  [static]      "Fast static website with CDN"
7. 🎮 Minecraft    [game]        "Vanilla Minecraft Java server"
8. 🎯 CS2          [game]        "Counter-Strike 2 dedicated server"

---

### Экран 5 — Template Page (Minecraft)

Хлебные крошки: Applications > App Store > Minecraft

Шапка:

- Иконка 56px + "Minecraft" H1 + badge [game] + "v1.20.4" (monospace, text-muted)
- Кнопка [Launch Minecraft →] справа
- Подзаголовок: "Vanilla Minecraft Java Edition server"

Две колонки:

Левая (60%) — описание без технического жаргона:
  "Host your own Minecraft server for you and your friends.
   Fully managed with automatic backups and easy configuration."

  Что включено (чеклист с ✓):
  ✓ Automatic world backups
  ✓ Whitelist management
  ✓ Server console access
  ✓ Custom game modes

  Available actions:
  • Restart — Restart the server process
  • Create Backup — Create a backup of world data
  • Restore Backup — Restore from a previous backup

Правая (40%) — карточка с метаданными:
  Maintained by: Community
  Latest version: 1.20.4
  Last updated: 1 week ago
  Min. storage: 10 GB

  Resources (три pill-а):
  💻 2 CPU   🧠 2 GB RAM   💾 10 GB Storage

---

### Экран 6 — Launch Form (Minecraft)

Хлебные крошки: Applications > App Store > Minecraft > Launch

Заголовок: "Launch Minecraft"
Подзаголовок: "Your server will be ready in about 3 minutes"

Форма (одна колонка, max-width 560px):

**Секция "Basic"**:

- App name*        placeholder "my-minecraft"
  hint: "Used as your app's identifier. Lowercase, no spaces."

- Environment*     select: production / staging
  hint: "Where your application will run"

**Секция "Server Settings"**:

- Server name      placeholder "My Minecraft Server"
  hint: "Display name shown in server list"

- Max players      select: 10 / 20 / 50 / 100
  hint: "Maximum concurrent players"

- Game mode        select: Survival / Creative / Adventure / Spectator

- Enable whitelist toggle (off по умолчанию)
  hint: "Only allow approved players to join"

**Секция "Resources"**:

- Storage size     select: 10 GB / 20 GB / 50 GB

Footer: [🚀 Launch Minecraft] [Cancel]
Hint под кнопками: "You can change these settings later from the app dashboard."

---

### Экран 7 — Environments

Хлебные крошки: Environments
Заголовок: "Environments"
Кнопка [+ Create Environment] справа от заголовка

2 карточки окружений:

1. **production** | Ready
   Resources: 3 nodes · 12 CPU · 48 GB RAM
   "Created 2 weeks ago"   [Manage] [⋮]

2. **staging** | Ready
   Resources: 2 nodes · 8 CPU · 32 GB RAM
   "Created 1 week ago"    [Manage] [⋮]

Под карточками: мелкий текст "2 environments · 2 ready"

---

### Экран 8 — Create Environment

Хлебные крошки: Environments > Create

Заголовок: "Create Environment"
Подзаголовок: "Provisioning takes about 5-10 minutes"

Форма (одна колонка, max-width 560px):

**Секция "Basic"**:

- Environment name*  placeholder "development"
  hint: "Used as identifier. Lowercase, no spaces."

**Секция "Resources" (optional)**:

Info box (bg-elevated, border-left 3px accent-blue):
"Leave empty for automatic sizing based on your workload.
 You can adjust resources later."

- Max nodes         input number, placeholder "auto"
  hint: "Maximum number of worker nodes"

- Node size         select: Auto / Small (2 CPU, 4 GB) / Medium (4 CPU, 8 GB) / Large (8 CPU, 16 GB)

Footer: [Create Environment] [Cancel]

---

## ТЕХНИЧЕСКИЕ ТРЕБОВАНИЯ

Используется Ant Design (antd v5) как базовая дизайн-система.
Все кастомизации — через AntD Design Tokens (ConfigProvider theme),
НЕ через перебивание CSS классов.

### Подключение (CDN, один HTML-файл):

```html
<script src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
<script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
<script src="https://unpkg.com/antd@5/dist/antd.min.js"></script>
<link  href="https://unpkg.com/antd@5/dist/reset.css" rel="stylesheet">
```

JSX компилировать через Babel Standalone или писать чистый React.createElement.

### ConfigProvider — переопределение токенов:

```js
const theme = {
  algorithm: antd.theme.darkAlgorithm,
  token: {
    colorPrimary:        '#4c7ef3',
    colorSuccess:        '#3ecf8e',
    colorWarning:        '#f5a623',
    colorError:          '#f87171',
    colorBgBase:         '#1a1a1e',
    colorBgContainer:    '#222226',
    colorBgElevated:     '#2a2a2e',
    colorBgLayout:       '#16161a',
    colorBorder:         'rgba(255,255,255,0.08)',
    colorBorderSecondary:'rgba(255,255,255,0.14)',
    colorText:           '#e8e8ea',
    colorTextSecondary:  '#9999a8',
    colorTextTertiary:   '#666672',
    borderRadius:        6,
    borderRadiusLG:      10,
    borderRadiusSM:      4,
    fontFamily:          '-apple-system, system-ui, sans-serif',
    fontSize:            13,
    lineWidth:           0.5,
  },
  components: {
    Layout: {
      siderBg:           '#16161a',
      headerBg:          '#16161a',
      bodyBg:            '#1a1a1e',
    },
    Menu: {
      darkItemBg:        '#16161a',
      darkSubMenuItemBg: '#16161a',
      darkItemSelectedBg:'#2a2a2e',
      itemHeight:        32,
      collapsedIconSize: 14,
    },
    Card: {
      paddingLG:         16,
    },
    Table: {
      headerBg:          '#222226',
      rowHoverBg:        '#2e2e34',
      borderColor:       'rgba(255,255,255,0.08)',
    },
    Button: {
      paddingInline:     14,
    },
    Input: {
      colorBgContainer:  '#2a2a2e',
    },
    Select: {
      colorBgContainer:  '#2a2a2e',
      colorBgElevated:   '#2a2a2e',
    },
    Modal: {
      contentBg:         '#222226',
      headerBg:          '#222226',
      footerBg:          '#222226',
    },
  },
};
```

### Маппинг компонентов AntD на макет:

- Layout, Layout.Sider, Layout.Header, Layout.Content — каркас страницы
- Menu (mode="inline", theme="dark") — боковая навигация
- Card — карточки приложений, environments и шаблонов
- Tag — category badges и status badges (color через colorBgContainer/color)
- Descriptions — таблица key-value в Application Details
- Form + Form.Item — формы запуска и создания
- Input, Input.Password, Select, Switch, InputNumber — поля формы
- Button (type="primary" / "default" / "text") — кнопки
- Dropdown — Actions dropdown в Application Details
- Modal — Action Modal
- Breadcrumb — хлебные крошки
- Badge + Tag для статусов
- Typography.Title, Typography.Text — заголовки и текст
- Space, Flex, Row, Col — лейаут внутри контента
- Divider — разделители секций формы
- Alert — info box в Create Environment

### Кастомный стиль поверх AntD (минимальный глобальный CSS):

Только то, чего нельзя добиться токенами:

- Скрыть иконки в Menu (нет встроенного токена)
- Логотип в Sider
- Разделители между экранами мока
- Цвета конкретных Tag-ов по категориям (cms/nodejs/php/static/game)
  — через inline style или className, не перебивая .ant-tag глобально
- Resource pills в Environment cards

---

## CRD MAPPING (для разработчиков)

Связь UI с Kubernetes ресурсами:

| UI Element | CRD | Field |
|------------|-----|-------|
| App name | Application | metadata.name |
| Environment selector | Application | spec.targetEnvironment |
| Template | Application | spec.templateRef |
| App values | Application | spec.values |
| App status | Application | status.phase |
| Template name | ApplicationTemplate | metadata.name |
| Template parameters | ApplicationTemplate | spec.parameters |
| Template actions | ApplicationTemplate | spec.actions |
| Action form | Action | spec.values |
| Action status | Action | status.phase |
| Environment name | Environment | metadata.name |
| Environment status | Environment | status.phase |
| Environment resources | Environment | status.resources |
| Environment sizing | Environment | spec.sizing |
