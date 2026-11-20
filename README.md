# Практична робота

**Тема:** Валідація вхідних даних, санітизація та логування запитів у Request Pipeline фреймворку NestJS  
**Варіант 3:** Одяг та взуття (`Clothing`)

---

## 1. Мета роботи

* Опанувати архітектуру життєвого циклу HTTP-запиту (**Request Pipeline**) у фреймворку **NestJS** та порядок взаємодії його складових (*Middleware* $\rightarrow$ *Guards* $\rightarrow$ *Interceptors* $\rightarrow$ *Pipes* $\rightarrow$ *Controller Route Handler* $\rightarrow$ *Exception Filters*).
* Навчитися проєктувати суворі контракти вхідних даних за допомогою класів **DTO** (*Data Transfer Object*) та бібліотек `class-validator` і `class-transformer`.
* Налаштувати глобальний `ValidationPipe` із фільтрацією білого списку властивостей (`whitelist`, `forbidNonWhitelisted`) та автоматичним приведенням типів (`transform`).
* Розробити власний пайп (**Custom Pipe** — `TrimPipe`) для рекурсивної санітизації рядкових значень у вхідних тілах запитів.
* Реалізувати та зареєструвати мережеве **Middleware** (`HttpLoggingMiddleware`) для моніторингу статусів і вимірювання затримки обробки HTTP-запитів у мілісекундах.

---

## 2. Теоретичні відомості

### 2.1. Архітектура Request Pipeline у NestJS

Кожен вхідний HTTP-запит у застосунку NestJS проходить через суворо визначений конвеєр обробки:

```text
HTTP Request
   │
   ▼
[ 1. Middleware ] ──────────────► Глобальні або модульні проміжні обробники (логування, CORS)
   │
   ▼
[ 2. Guards ] ──────────────────► Перевірка дозволів та автентифікації (RBAC, JWT)
   │
   ▼
[ 3. Interceptors (Before) ] ───► Перехоплення запиту до обробника (бенчмаркінг, кеш)
   │
   ▼
[ 4. Pipes ] ───────────────────► Санітизація, трансформація та валідація вхідних даних (DTO)
   │
   ▼
[ 5. Controller Route Handler ] ─► Виконання бізнес-логіки та звернення до сервісів
   │
   ▼
[ 6. Interceptors (After) ] ────► Модифікація вихідної відповіді (серіалізація, заголовки)
   │
   ▼
[ 7. Exception Filters ] ───────► Перехоплення невідловлених виключень та формування JSON-помилок
   │
   ▼
HTTP Response
```

Така послідовність гарантує, що **Middleware** фіксує абсолютно всі запити (навіть ті, що згодом будуть відхилені на етапі валідації), а **Pipes** виконуються безпосередньо перед викликом методу контролера, захищаючи бізнес-логіку від некоректних або небезпечних даних.

### 2.2. Декларативна валідація та санітизація

* **`class-transformer`** перетворює сирий JavaScript-об'єкт, отриманий із тіла JSON-запиту, на повноцінний екземпляр класу DTO.
* **`class-validator`** перевіряє властивості створеного екземпляра згідно з декларативними декораторами (`@IsString()`, `@IsNumber()`, `@IsEnum()`, `@IsPositive()` тощо).
* **Послідовність пайпів:** при реєстрації кількох глобальних пайпів вони виконуються у порядку передачі до `app.useGlobalPipes(...)`. Розміщення `TrimPipe` перед `ValidationPipe` гарантує, що рядки, які складаються виключно з пробілів (наприклад, `"   "`), спочатку перетворяться на порожній рядок `""`, а потім будуть коректно відхилені декоратором `@IsNotEmpty()`.

---

## 3. Технологічний стек та структура проєкту

* **Середовище виконання та мова:** Node.js, TypeScript (CommonJS)
* **Фреймворк:** NestJS (`@nestjs/common`, `@nestjs/core`, `@nestjs/platform-express`)
* **Валідація та трансформація:** `class-validator`, `class-transformer`
* **Логування:** Вбудований клас `Logger` із `@nestjs/common`

```text
pr-nestjs-validation/
├── src/
│   ├── clothing/
│   │   ├── dto/
│   │   │   ├── create-clothing.dto.ts      # Контракт створення позиції одягу/взуття
│   │   │   └── update-clothing.dto.ts      # Контракт часткового оновлення (@IsOptional)
│   │   ├── entities/
│   │   │   └── clothing.entity.ts          # Сутність ресурсу Clothing
│   │   ├── clothing.controller.ts          # REST-контролер маршрутів /clothing
│   │   ├── clothing.module.ts              # Функціональний модуль ресурсу
│   │   └── clothing.service.ts             # Сервіс бізнес-логіки
│   ├── common/
│   │   ├── middleware/
│   │   │   └── http-logger.middleware.ts   # Middleware логування HTTP-трафіку та затримки
│   │   └── pipes/
│   │       └── trim.pipe.ts                # Custom Pipe рекурсивного очищення пробілів
│   ├── app.controller.ts
│   ├── app.module.ts                       # Кореневий модуль (реєстрація Middleware)
│   ├── app.service.ts
│   └── main.ts                             # Точка входу (реєстрація глобальних Pipes)
├── package.json
├── tsconfig.json
└── README.md                               # Звіт до практичної роботи
```

---

## 4. Хід виконання та реалізація програмного коду

### 4.1. Рівень 1. Базовий: Підключення `class-validator` та контракти DTO

#### Налаштування глобального конвеєра валідації (`src/main.ts`)
У точці входу застосунку зареєстровано глобальний `TrimPipe` та `ValidationPipe` із суворими параметрами безпеки (`whitelist: true`, `forbidNonWhitelisted: true`, `transform: true`):

```typescript
import { NestFactory } from '@nestjs/core';
import { ValidationPipe } from '@nestjs/common';
import { AppModule } from './app.module';
import { TrimPipe } from './common/pipes/trim.pipe';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.useGlobalPipes(
    new TrimPipe(),
    new ValidationPipe({
      whitelist: true,
      forbidNonWhitelisted: true,
      transform: true,
    }),
  );

  await app.listen(3000);
}
bootstrap();
```

#### Контракт створення ресурсу (`src/clothing/dto/create-clothing.dto.ts`)
Для **Варіанту 3 (Одяг та взуття)** визначено модель із 5 обов'язкових полів (`title`, `price`, `size`, `material`, `color`) та перерахування допустимих розмірів `ClothingSize`:

```typescript
import {
  IsString,
  IsNotEmpty,
  IsNumber,
  IsPositive,
  IsEnum,
  MinLength,
  MaxLength,
} from 'class-validator';

export enum ClothingSize {
  XS = 'XS',
  S = 'S',
  M = 'M',
  L = 'L',
  XL = 'XL',
}

export class CreateClothingDto {
  @IsString({ message: 'Назва товару (title) має бути рядком' })
  @IsNotEmpty({ message: 'Назва товару є обов’язковою' })
  @MinLength(2, { message: 'Мінімальна довжина назви — 2 символи' })
  @MaxLength(100, { message: 'Максимальна довжина назви — 100 символів' })
  readonly title: string;

  @IsNumber({}, { message: 'Ціна (price) має бути числом' })
  @IsPositive({ message: 'Ціна має бути додатним числом більше 0' })
  readonly price: number;

  @IsEnum(ClothingSize, {
    message: 'Розмір (size) має бути одним із значень: XS, S, M, L, XL',
  })
  readonly size: ClothingSize;

  @IsString({ message: 'Матеріал (material) має бути рядком' })
  @IsNotEmpty({ message: 'Матеріал є обов’язковим полем' })
  readonly material: string;

  @IsString({ message: 'Колір (color) має бути рядком' })
  @IsNotEmpty({ message: 'Колір є обов’язковим полем' })
  readonly color: string;
}
```

#### Контракт оновлення ресурсу (`src/clothing/dto/update-clothing.dto.ts`)
У `UpdateClothingDto` кожна властивість позначена декоратором `@IsOptional()`, що дозволяє передавати лише змінювані поля у `PATCH`-запитах, зберігаючи повну перевірку типів і обмежень:

```typescript
import {
  IsString,
  IsNotEmpty,
  IsNumber,
  IsPositive,
  IsEnum,
  IsOptional,
  MinLength,
  MaxLength,
} from 'class-validator';
import { ClothingSize } from './create-clothing.dto';

export class UpdateClothingDto {
  @IsOptional()
  @IsString({ message: 'Назва товару (title) має бути рядком' })
  @IsNotEmpty({ message: 'Назва товару не може бути порожньою' })
  @MinLength(2, { message: 'Мінімальна довжина назви — 2 символи' })
  @MaxLength(100, { message: 'Максимальна довжина назви — 100 символів' })
  readonly title?: string;

  @IsOptional()
  @IsNumber({}, { message: 'Ціна (price) має бути числом' })
  @IsPositive({ message: 'Ціна має бути додатним числом більше 0' })
  readonly price?: number;

  @IsOptional()
  @IsEnum(ClothingSize, {
    message: 'Розмір (size) має бути одним із значень: XS, S, M, L, XL',
  })
  readonly size?: ClothingSize;

  @IsOptional()
  @IsString({ message: 'Матеріал (material) має бути рядком' })
  @IsNotEmpty({ message: 'Матеріал не може бути порожнім' })
  readonly material?: string;

  @IsOptional()
  @IsString({ message: 'Колір (color) має бути рядком' })
  @IsNotEmpty({ message: 'Колір не може бути порожнім' })
  readonly color?: string;
}
```

---

### 4.2. Рівень 2. Середній: Санітизація даних та мережеве Middleware

#### Рекурсивний пайп очищення рядків (`src/common/pipes/trim.pipe.ts`)
`TrimPipe` реалізує інтерфейс `PipeTransform` і рекурсивно обходить вхідні значення будь-якої вкладеності (окремі рядки, масиви, об'єкти), видаляючи початкові та кінцеві пробіли:

```typescript
import { PipeTransform, Injectable, ArgumentMetadata } from '@nestjs/common';

@Injectable()
export class TrimPipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata) {
    return this.trimRecursive(value);
  }

  private trimRecursive(value: any): any {
    if (typeof value === 'string') {
      return value.trim();
    }

    if (Array.isArray(value)) {
      return value.map((item) => this.trimRecursive(item));
    }

    if (typeof value === 'object' && value !== null) {
      const trimmedObj: Record<string, any> = {};
      for (const key of Object.keys(value)) {
        trimmedObj[key] = this.trimRecursive(value[key]);
      }
      return trimmedObj;
    }

    return value;
  }
}
```

#### Middleware логування HTTP-запитів (`src/common/middleware/http-logger.middleware.ts`)
`HttpLoggingMiddleware` фіксує момент надходження запиту, підписується на подію `finish` об'єкта відповіді та виводить у консоль метод, URL, код статусу, затримку в мілісекундах та IP-адресу клієнта:

```typescript
import { Injectable, NestMiddleware, Logger } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class HttpLoggingMiddleware implements NestMiddleware {
  private readonly logger = new Logger('HTTP');

  use(req: Request, res: Response, next: NextFunction) {
    const { method, originalUrl, ip } = req;
    const startTime = Date.now();

    res.on('finish', () => {
      const { statusCode } = res;
      const duration = Date.now() - startTime;
      this.logger.log(
        `${method} ${originalUrl} ${statusCode} +${duration}ms [${ip}]`,
      );
    });

    next();
  }
}
```

#### Реєстрація Middleware у кореневому модулі (`src/app.module.ts`)

```typescript
import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ClothingModule } from './clothing/clothing.module';
import { HttpLoggingMiddleware } from './common/middleware/http-logger.middleware';

@Module({
  imports: [ClothingModule],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer.apply(HttpLoggingMiddleware).forRoutes('*');
  }
}
```

---

## 5. Результати тестування API

### 5.1. Сценарій 1 — Валідний запит із зайвими пробілами (перевірка `TrimPipe`)

**Запит:**
```powershell
curl.exe -X POST http://localhost:3000/clothing -H "Content-Type: application/json" -d '{\"title\": \"   Кросівки бігові   \", \"price\": 3200, \"size\": \"L\", \"material\": \"  Шкіра  \", \"color\": \" Білий \"}'
```

**Отримана відповідь (`201 Created`):**
```text
This action adds a new clothing
```
Усі рядкові поля успішно пройшли санітизацію через `TrimPipe` (`"Кросівки бігові"`, `"Шкіра"`, `"Білий"`) та задовольнили умови `CreateClothingDto`.

---

### 5.2. Сценарій 2 — Некоректні дані та заборонені поля (перевірка `ValidationPipe`)

**Запит:**
```powershell
curl.exe -X POST http://localhost:3000/clothing -H "Content-Type: application/json" -d '{\"title\": \"\", \"price\": -150, \"size\": \"XXL\", \"material\": \"   \", \"color\": \"Чорний\", \"extraField\": 123}'
```

**Отримана відповідь (`400 Bad Request`):**
```json
{
  "message": [
    "property extraField should not exist",
    "Мінімальна довжина назви — 2 символи",
    "Назва товару є обов’язковою",
    "Ціна має бути додатним числом більше 0",
    "Розмір (size) має бути одним із значень: XS, S, M, L, XL",
    "Матеріал є обов’язковим полем"
  ],
  "error": "Bad Request",
  "statusCode": 400
}
```

**Аналіз результату:**
* Параметр `forbidNonWhitelisted: true` заблокував невідоме поле `extraField`.
* `TrimPipe` обрізав пробіли у полі `material: "   "` до порожнього рядка `""`, завдяки чому спрацював декоратор `@IsNotEmpty()`.
* Валідатори `@MinLength`, `@IsPositive` та `@IsEnum` сформували вичерпний масив людиночитаних повідомлень про всі порушення контракту.

---

### 5.3. Логування HTTP-трафіку на стороні сервера (`HttpLoggingMiddleware`)

![1](/pr-nestjs-validation/image/1.png)

---
![2](/pr-nestjs-validation/image/image.png)

## 6. Висновки

У ході виконання практичної роботи було досліджено архітектуру **Request Pipeline** фреймворку NestJS та реалізовано повноцінний механізм захисту, санітизації та моніторингу REST API для ресурсу «Одяг та взуття» (`Clothing`):

1. Використання декларативних DTO із бібліотеками `class-validator` та `class-transformer` у поєднанні з глобальним `ValidationPipe` повністю ізолює контролери та сервіси від невалідних типів даних і небажаних властивостей (`whitelist`, `forbidNonWhitelisted`).
2. Розроблений `TrimPipe` автоматизує попереднє очищення вхідних рядків на рівні всього застосунку без дублювання логіки в окремих обробниках маршрутів.
3. Впровадження `HttpLoggingMiddleware` на початковому етапі конвеєра забезпечує централізований аудит усіх вхідних запитів, їхніх HTTP-статусів та часу виконання.