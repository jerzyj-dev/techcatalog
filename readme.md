# Техническая спецификация (PRD): TechCatalog (Производственная версия)

## 1. Архитектура приложения
Приложение строится по классической трехслойной асинхронной архитектуре Monolith-ready SPA:
* **Data Layer (PostgreSQL 18+):** Хранит реляционные данные, иерархические структуры изделий (BOM), историю статусов, пользователей и матрицы прав доступа.
* **API Layer (FastAPI + Python 3.14+):** Асинхронный бэкенд (драйвер asyncpg, ORM SQLAlchemy 2.1). Обеспечивает авторизацию на базе безопасных JWT в HTTP-only Cookie, валидацию данных через Pydantic v2 и бизнес-логику проверок прав доступа.
* **Presentation Layer (Vue 3 + Pinia + Чистый CSS):** Одностраничное приложение (SPA). Загружает массивы данных в UI-стейт (до 2000 строк для комфортного рендеринга без Virtual Scroll), реализует Excel-подобную фильтрацию и сортировку на клиенте. Стили полностью изолированы внутри SFC-компонентов через атрибут scoped.

---

## 2. Структура базы данных PostgreSQL

### 2.1. Схема Core-сущностей (Оргструктура и Пользователи)
```sql
-- Справочник должностей
CREATE TABLE positions (
    id SERIAL PRIMARY KEY,
    title VARCHAR(150) NOT NULL UNIQUE
);

-- Справочник отделов (Древовидная иерархическая структура)
CREATE TABLE departments (
    id SERIAL PRIMARY KEY,
    name VARCHAR(150) NOT NULL UNIQUE,
    parent_id INTEGER REFERENCES departments(id) ON DELETE SET NULL
);

-- Таблица Пользователей (с фото и оргструктурой)
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    hashed_password VARCHAR(255) NOT NULL,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    phone VARCHAR(20),
    birth_date DATE,
    hire_date DATE,
    avatar_url VARCHAR(512),
    is_admin BOOLEAN DEFAULT FALSE,
    is_department_leader BOOLEAN DEFAULT FALSE,
    position_id INTEGER REFERENCES positions(id) ON DELETE RESTRICT,
    department_id INTEGER REFERENCES departments(id) ON DELETE RESTRICT
);

-- Матрица прав доступа к бизнес-таблицам (ACL на уровне таблиц)
CREATE TABLE table_permissions (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    table_name VARCHAR(100) NOT NULL,
    can_view BOOLEAN DEFAULT TRUE,
    can_edit BOOLEAN DEFAULT FALSE,
    UNIQUE(user_id, table_name)
);
```

### 2.2. Схема иерархии изделий (BOM) и производственных заказов
```sql
CREATE TABLE projects (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    budget NUMERIC(12, 2),
    deadline DATE
);

CREATE TABLE modules (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL
);

CREATE TABLE details (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    drawing_number VARCHAR(100)
);

CREATE TABLE project_modules (
    project_id INTEGER REFERENCES projects(id) ON DELETE CASCADE,
    module_id INTEGER REFERENCES modules(id) ON DELETE CASCADE,
    PRIMARY KEY (project_id, module_id)
);

CREATE TABLE project_details (
    project_id INTEGER REFERENCES projects(id) ON DELETE CASCADE,
    detail_id INTEGER REFERENCES details(id) ON DELETE CASCADE,
    PRIMARY KEY (project_id, detail_id)
);

CREATE TABLE module_submodules (
    parent_module_id INTEGER REFERENCES modules(id) ON DELETE CASCADE,
    child_module_id INTEGER REFERENCES modules(id) ON DELETE CASCADE,
    PRIMARY KEY (parent_module_id, child_module_id)
);

CREATE TABLE module_details (
    module_id INTEGER REFERENCES modules(id) ON DELETE CASCADE,
    detail_id INTEGER REFERENCES details(id) ON DELETE CASCADE,
    PRIMARY KEY (module_id, detail_id)
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    order_number VARCHAR(100) NOT NULL UNIQUE,
    status VARCHAR(100) DEFAULT 'Новый' NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP NOT NULL
);
```

### 2.3. Схема Технической Оснастки
```sql
CREATE TABLE tooling (
    id SERIAL PRIMARY KEY,
    designation VARCHAR(100) NOT NULL UNIQUE,
    name VARCHAR(255) NOT NULL,
    current_status VARCHAR(100) DEFAULT 'Проектирование' NOT NULL,
    description TEXT,
    comment TEXT
);

CREATE INDEX idx_tooling_designation ON tooling(designation);
CREATE INDEX idx_orders_number ON orders(order_number);

CREATE TABLE tooling_assignees (
    tooling_id INTEGER REFERENCES tooling(id) ON DELETE CASCADE,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    PRIMARY KEY (tooling_id, user_id)
);

CREATE TABLE tooling_projects (
    tooling_id INTEGER REFERENCES tooling(id) ON DELETE CASCADE,
    project_id INTEGER REFERENCES projects(id) ON DELETE CASCADE,
    PRIMARY KEY (tooling_id, project_id)
);

CREATE TABLE tooling_modules (
    tooling_id INTEGER REFERENCES tooling(id) ON DELETE CASCADE,
    module_id INTEGER REFERENCES modules(id) ON DELETE CASCADE,
    PRIMARY KEY (tooling_id, module_id)
);

CREATE TABLE tooling_details (
    tooling_id INTEGER REFERENCES tooling(id) ON DELETE CASCADE,
    detail_id INTEGER REFERENCES details(id) ON DELETE CASCADE,
    PRIMARY KEY (tooling_id, detail_id)
);

CREATE TABLE order_tooling (
    order_id INTEGER REFERENCES orders(id) ON DELETE CASCADE,
    tooling_id INTEGER REFERENCES tooling(id) ON DELETE CASCADE,
    PRIMARY KEY (order_id, tooling_id)
);

CREATE TABLE tooling_status_history (
    id SERIAL PRIMARY KEY,
    tooling_id INTEGER REFERENCES tooling(id) ON DELETE CASCADE,
    status VARCHAR(100) NOT NULL,
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP NOT NULL,
    changed_by_user_id INTEGER REFERENCES users(id) ON DELETE SET NULL
);
```

---

## 3. API Спецификация (FastAPI)
Сессия и авторизация проверяются через передачу токена в HTTP-only Cookie `access_token`.

| Метод | URL | Описание | Ограничение доступа |
| :--- | :--- | :--- | :--- |
| **POST** | `/api/v1/auth/login` | Аутентификация, выдача HTTP-only Cookies | Доступно всем |
| **POST** | `/api/v1/admin/users` | Регистрация нового пользователя админом | Только `is_admin = True` |
| **POST** | `/api/v1/users/{id}/avatar` | Загрузка фото пользователя | Только админ или сам юзер |
| **GET** | `/api/v1/data/{table_name}` | Чтение всех строк таблицы | Проверка ACL (`can_view`) |
| **POST** | `/api/v1/data/{table_name}` | Создание записи | Проверка ACL (`can_edit`) |
| **PUT** | `/api/v1/data/{table_name}/{id}` | Обновление записи | Проверка ACL (`can_edit`) |

---

## 4. Архитектура Фронтенда (Vue 3)

### 4.1. Глобальные стили (`src/assets/variables.css`)
```css
:root {
  --bg-primary: #ffffff;
  --bg-excel-header: #f8f9fa;
  --border-color: #e1e4e8;
  --text-main: #333333;
  --accent-excel: #107c41;
  --accent-hover: #0f6f3a;
}
```

### 4.2. Пример Vue-компонента таблицы с фильтрами (`src/components/ExcelTable.vue`)
```vue
<script setup>
import { ref, computed } from 'vue'
const props = defineProps({
  columns: { type: Array, required: true },
  rows: { type: Array, required: true }
})
const emit = defineEmits(['openCreateModal', 'copyRow'])
const searchQueries = ref({})
const activeFilterDropdown = ref(null)
const toggleFilter = (col) => {
  activeFilterDropdown.value = activeFilterDropdown.value === col ? null : col
}
const filteredRows = computed(() => {
  return props.rows.filter(row => {
    return Object.keys(searchQueries.value).every(col => {
      const query = searchQueries.value[col]?.toLowerCase()
      if (!query) return true
      return String(row[col]).toLowerCase().includes(query)
    })
  })
})
</script>

<template>
  <div class="table-container">
    <table class="excel-table">
      <thead>
        <tr>
          <th v-for="col in columns" :key="col" class="excel-th">
            <div class="header-cell">
              <span>{{ col.toUpperCase() }}</span>
              <button @click="toggleFilter(col)">🔽</button>
            </div>
            <div v-if="activeFilterDropdown === col" class="filter-dropdown">
              <input v-model="searchQueries[col]" placeholder="Поиск..." class="filter-input" />
            </div>
          </th>
        </tr>
      </thead>
      <tbody>
        
          
            
              
            
            {{ row[col] }}
          
        
      
    
  



.excel-table { width: 100%; border-collapse: collapse; }
.excel-th, .excel-td { border: 1px solid var(--border-color); padding: 6px 10px; }
.user-avatar-mini { width: 28px; height: 28px; border-radius: 50%; object-fit: cover; }

```
