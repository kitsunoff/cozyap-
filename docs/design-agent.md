Ты дизайнер интерфейсов. Создай HTML-мок нового раздела "My Applications"
в Cozystack — self-hosted PaaS для провайдеров (ISP).

Целевой пользователь: клиент провайдера, нетехнический. Он не знает что такое
Kubernetes, namespace, YAML. Хочет просто запустить WordPress или Node.js-сайт
как в Heroku/Vercel — выбрал, настроил, нажал кнопку.

---

## СЦЕНАРИЙ (4 экрана, один long-scroll HTML)

1. My Applications — список запущенных приложений с их статусами
2. App Store — каталог шаблонов для запуска нового
3. Страница шаблона — описание, требования, кнопка "Launch"
4. Форма запуска — простые человекочитаемые поля, без технического жаргона

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
- Секции: Applications / Resources / Backups / Settings
- "Applications" раскрыта: My Apps (активный), App Store
- Активный пункт: bg-elevated, font-weight 500
- Hover: bg-hover

**Status badge** (height 22px, padding 0 8px, border-radius 5px, font-size 11px):
- Running:  bg rgba(62,207,142,0.15),  color #3ecf8e
- Starting: bg rgba(245,166,35,0.15),  color #f5a623
- Stopped:  bg rgba(255,255,255,0.06), color #9999a8
- Error:    bg rgba(248,113,113,0.15), color #f87171

**Category badge** (height 20px, padding 0 7px, border-radius 4px, font-size 11px):
- cms:     bg rgba(52,130,246,0.15),  color #6aaaf8
- nodejs:  bg rgba(62,207,142,0.15),  color #5dd6a0
- php:     bg rgba(144,97,249,0.15),  color #b08af9
- static:  bg rgba(245,166,35,0.15),  color #f5c462

**App-карточка в "My Apps"** (bg-surface, border 0.5px, border-radius 10px, padding 16px):
- Строка 1: иконка (32px) + название приложения + status badge справа
- Строка 2: тип шаблона (12px, text-muted) + домен (12px, accent-blue, как ссылка)
- Строка 3: "Launched 3 days ago" + кнопки [Open ↗] [Manage] [⋮]
- Hover: border-color → border-hover

**Каталог-карточка в "App Store"** (bg-surface, border 0.5px, border-radius 10px, padding 16px):
- Иконка: 44×44px, border-radius 10px, цветной bg opacity 0.12
- Название: 14px, font-weight 500
- Category badge под названием
- Описание: 12px, text-secondary, 2 строки max
- Кнопка [Launch →] в правом нижнем углу

**Форма запуска** (без YAML, без технических терминов):
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

**Разделители**: 0.5px solid rgba(255,255,255,0.08) везде

---

## КОНТЕНТ ПО ЭКРАНАМ

### Экран 1 — My Applications

Хлебные крошки: Applications
Заголовок: "My Applications"
Кнопка [+ Launch New App] справа от заголовка

3 карточки запущенных приложений:

1. 🌐 my-blog          | WordPress  | Running  | my-blog.example.com
   "WordPress · Launched 5 days ago"    [Open ↗] [Manage] [⋮]

2. 🛒 shop             | WooCommerce| Starting | shop.example.com
   "WooCommerce · Launched 2 minutes ago"  [Open ↗] [Manage] [⋮]

3. 📄 landing          | Static Site| Stopped  | landing.example.com
   "Static Site · Launched 12 days ago"  [Start] [Manage] [⋮]

Под карточками: мелкий текст "3 applications · 2 running"

### Экран 2 — App Store

Хлебные крошки: Applications > App Store
Заголовок: "App Store"
Строка поиска + фильтры: All / CMS / Node.js / PHP / Static

Сетка карточек (3 колонки, 6 штук):

1. 🔵 WordPress    [cms]         "The world's most popular CMS"
2. 🟣 Drupal       [cms][php]    "Enterprise content management"
3. 👻 Ghost        [cms]         "Modern publishing platform"
4. 🟢 Node.js App  [nodejs]      "Deploy any Node.js application"
5. ⚡ Next.js      [nodejs]      "Full-stack React framework"
6. 📄 Static Site  [static]      "Fast static website with CDN"

### Экран 3 — Страница шаблона (WordPress)

Хлебные крошки: Applications > App Store > WordPress

Шапка:
- Иконка 56px + "WordPress" H1 + badge [cms] + "v6.4" (monospace, text-muted)
- Кнопка [Launch WordPress →] справа
- Подзаголовок: "The world's most popular content management system"

Две колонки:
Левая (60%) — описание без технического жаргона:
  "Perfect for blogs, business websites, and online stores.
   Includes automatic updates, daily backups, and SSL certificate."

  Что включено (чеклист с ✓):
  ✓ Automatic WordPress updates
  ✓ Daily backups
  ✓ Free SSL certificate
  ✓ Admin dashboard at /wp-admin

Правая (40%) — карточка с метаданными:
  Maintained by: Cozystack Team
  Latest version: 6.4.2
  Last updated: 2 weeks ago
  Min. storage: 5 GB

  Resources (три pill-а):
  💻 1 CPU   🧠 512 MB RAM   💾 5 GB Storage

### Экран 4 — Форма запуска WordPress

Хлебные крошки: Applications > App Store > WordPress > Launch

Заголовок: "Launch WordPress"
Подзаголовок: "Your app will be ready in about 2 minutes"

Форма (одна колонка, max-width 560px):

Секция "Basic":
- App name*        placeholder "my-wordpress"
  hint: "Used as your app's identifier. Lowercase, no spaces."
- Your domain      placeholder "myblog.example.com"
  hint: "Leave empty to use auto-generated domain"

Секция "WordPress Settings":
- Admin email* 🔒   placeholder "you@example.com"
- Admin password* 🔒 type=password
- Site title        placeholder "My Awesome Blog"

Секция "Resources":
- Storage size   select: 5 GB / 10 GB / 20 GB / 50 GB
- Enable cache   toggle (on по умолчанию)
  hint: "Speeds up your site with page caching"

Кнопки: [🚀 Launch WordPress] [Cancel]
Hint под кнопками: "You can change these settings later from the app dashboard."

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
  },
};
```

### Маппинг компонентов AntD на макет:
- Layout, Layout.Sider, Layout.Header, Layout.Content — каркас страницы
- Menu (mode="inline", theme="dark") — боковая навигация
- Card — карточки приложений и шаблонов
- Tag — category badges и status badges (color через colorBgContainer/color)
- Table — список запущенных приложений (альтернатива карточкам)
- Form + Form.Item — форма запуска
- Input, Input.Password, Select, Switch, InputNumber — поля формы
- Button (type="primary" / "default" / "text") — кнопки
- Breadcrumb — хлебные крошки
- Badge + Tag для статусов Running/Starting/Stopped/Error
- Typography.Title, Typography.Text — заголовки и текст
- Space, Flex, Row, Col — лейаут внутри контента
- Divider — разделители секций формы

### Кастомный стиль поверх AntD (минимальный глобальный CSS):
Только то, чего нельзя добиться токенами:
- Скрыть иконки в Menu (нет встроенного токена)
- Логотип в Sider
- Разделители между экранами мока
- Цвета конкретных Tag-ов по категориям (cms/nodejs/php/static)
  — через inline style или className, не перебивая .ant-tag глобально