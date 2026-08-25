---
title: Angular 與 .NET Core 的依賴注入(DI)比較筆記_ChatGPT
date: 2026-08-24 16:15:39
categories:
 - 前端開發
tags:
 - Angular
 - AI產出
---
<!-- # Angular DI、Interface 與 InjectionToken 實務整理 -->
這篇筆記以熟悉 .NET Core DI 的開發者角度，整理 Angular Dependency Injection（DI）、Interface、InjectionToken，以及實際如何替換 Service 實作。
<span style="color:red;">文章內容透過ai整理產出</span>
<!--more-->
---

## 1. Angular 也有 Dependency Injection（DI）

Dependency Injection（依賴注入）的核心概念是：

> 類別需要使用其他物件時，不自己 `new` 該物件，而是交由 DI Container 建立並注入。

例如：

```typescript
export class UserComponent {

  constructor(
    private userService: UserService
  ) {}

}
```

Component 不需要：

```typescript
const service = new UserService();
```

而是：

```text
UserComponent
      ↓
「我需要 UserService」
      ↓
Angular DI Container
      ↓
建立 / 取得 UserService
      ↓
注入 UserComponent
```

---

# 2. Angular DI 通常在哪裡設定？

Angular 常見的 Provider 設定方式包括：

```typescript
@Injectable({
  providedIn: 'root'
})
export class UserService {
}
```

也可以在 Component：

```typescript
@Component({
  selector: 'app-user',
  providers: [
    UserService
  ]
})
export class UserComponent {
}
```

Unit Test 則常透過 `TestBed` 設定：

```typescript
TestBed.configureTestingModule({
  providers: [
    ...
  ]
});
```

因此 Angular 的 DI 設定不像 .NET Core 主要集中在 `Program.cs`，而是會依照 Injector scope 與使用情境分散在不同位置。

---

# 3. .NET Core 與 Angular DI 的差異

.NET Core 很常使用：

```csharp
public interface IUserService
{
    User GetUser();
}

public class UserService : IUserService
{
    public User GetUser()
    {
        ...
    }
}
```

然後：

```csharp
builder.Services.AddScoped<IUserService, UserService>();
```

Controller：

```csharp
public UserController(IUserService userService)
{
    _userService = userService;
}
```

概念：

```text
IUserService
      ↓
.NET DI Container
      ↓
UserService
      ↓
UserController
```

因此 .NET Core 很常見：

```text
Interface
    ↓
Implementation
    ↓
DI
```

Angular 也能做到類似設計，但有一個重要差異：

> **TypeScript 的 Interface 在編譯成 JavaScript 後會消失。**

---

# 4. 為什麼 Angular 不能直接使用 Interface 做 DI？

例如：

```typescript
export interface IUserService {
  getUser(): User;
}
```

這個 Interface 主要是給 TypeScript 在編譯階段做型別檢查。

編譯後概念上：

```typescript
interface IUserService {
  getUser(): User;
}

class UserService implements IUserService {
  getUser() {
    ...
  }
}
```

會變成類似：

```javascript
class UserService {
  getUser() {
    ...
  }
}
```

`IUserService` 本身不存在於 JavaScript Runtime。

因此 Angular 無法直接：

```typescript
@Inject(IUserService)
```

因為 Runtime 根本沒有 `IUserService` 這個物件可以拿來辨識。

---

# 5. Angular 的 DI Token

Angular DI 需要一個 Runtime 中仍然存在的東西，作為「識別某個依賴」的 Token。

如果直接使用 Class：

```typescript
@Injectable({
  providedIn: 'root'
})
export class UserService {
}
```

那麼：

```typescript
constructor(
  private userService: UserService
) {}
```

Angular 可以直接使用 `UserService` 這個 Class 作為 DI Token。

但是如果希望使用：

```text
Interface
    ↓
Implementation
```

就可以搭配 `InjectionToken`。

---

# 6. Interface + InjectionToken + DI

下面用一個 Storage Service 作為完整範例。

需求：

```text
UserComponent
      ↓
需要 Storage Service
```

但希望未來可以替換：

```text
LocalStorageService
SessionStorageService
MockStorageService
```

而 `UserComponent` 不需要知道底層實作。

---

# 7. 第一步：建立 Interface

建立：

```text
storage.service.interface.ts
```

```typescript
export interface StorageService {
  get(key: string): string | null;
  set(key: string, value: string): void;
}
```

這份 Interface 就是一份「契約」。

它規定：

```text
StorageService 必須提供：

get(string) → string | null

set(string, string) → void
```

---

## 7.1 `get()` 的意思

```typescript
get(key: string): string | null;
```

代表：

- 方法名稱：`get`
- `key` 必須是 `string`
- 回傳值可以是 `string`
- 也可以是 `null`

例如：

```typescript
storage.get('username');
```

可能得到：

```text
WeiHao
```

或：

```text
null
```

---

## 7.2 `set()` 的意思

```typescript
set(key: string, value: string): void;
```

代表：

- 方法名稱：`set`
- `key` 必須是 `string`
- `value` 必須是 `string`
- `void` 表示不回傳資料

例如：

```typescript
storage.set('username', 'WeiHao');
```

---

# 8. 第二步：建立 LocalStorageService

建立：

```text
local-storage.service.ts
```

```typescript
import { Injectable } from '@angular/core';
import { StorageService } from './storage.service.interface';

@Injectable()
export class LocalStorageService implements StorageService {

  get(key: string): string | null {
    return localStorage.getItem(key);
  }

  set(key: string, value: string): void {
    localStorage.setItem(key, value);
  }

}
```

這裡：

```typescript
implements StorageService
```

代表：

> `LocalStorageService` 必須符合 `StorageService` 的規格。

例如如果漏掉：

```typescript
set(...)
```

TypeScript 就會報錯。

---

# 9. 第三步：建立另一個實作

建立：

```text
session-storage.service.ts
```

```typescript
import { Injectable } from '@angular/core';
import { StorageService } from './storage.service.interface';

@Injectable()
export class SessionStorageService implements StorageService {

  get(key: string): string | null {
    return sessionStorage.getItem(key);
  }

  set(key: string, value: string): void {
    sessionStorage.setItem(key, value);
  }

}
```

現在有：

```text
                 StorageService
                    Interface
                       │
              ┌────────┴────────┐
              ↓                 ↓
   LocalStorageService   SessionStorageService
```

兩個 Service 都遵守同一份 Interface。

---

# 10. 第四步：建立 `storage.token.ts`

建立：

```text
storage.token.ts
```

內容只有：

```typescript
import { InjectionToken } from '@angular/core';
import { StorageService } from './storage.service.interface';

export const STORAGE_SERVICE =
  new InjectionToken<StorageService>('STORAGE_SERVICE');
```

這個檔案只有這幾行是正常的。

因為它的工作非常單純：

> **建立一個 Angular DI 使用的 Token。**

它不是 Service，也不是 Interface，所以不需要寫 Service 的實作邏輯。

---

# 11. 為什麼 `storage.token.ts` 只有一個 Token？

這一行：

```typescript
export const STORAGE_SERVICE =
  new InjectionToken<StorageService>('STORAGE_SERVICE');
```

可以拆成三個概念。

### ① `new InjectionToken<StorageService>`

建立 Angular 的 DI Token。

```text
InjectionToken
      ↓
Angular DI 使用的識別物
```

---

### ② `<StorageService>`

這是 TypeScript 的型別資訊。

```typescript
InjectionToken<StorageService>
```

主要是告訴 TypeScript：

> 這個 Token 對應的型別是 `StorageService`。

它不是告訴 Angular「我要建立哪個 Service」。

---

### ③ `'STORAGE_SERVICE'`

這是 Token 的名稱：

```typescript
'STORAGE_SERVICE'
```

主要是方便辨識與除錯。

真正決定實際使用哪一個 Service 的，是後面的 Provider：

```typescript
{
  provide: STORAGE_SERVICE,
  useClass: LocalStorageService
}
```

---

# 12. Interface 與 InjectionToken 的差別

這兩個非常容易混淆。

## Interface

```typescript
export interface StorageService {
  get(key: string): string | null;
  set(key: string, value: string): void;
}
```

用途：

> 定義「規格」。

它告訴 TypeScript：

```text
StorageService
├── get()
└── set()
```

而且編譯成 JavaScript 後會消失。

---

## InjectionToken

```typescript
export const STORAGE_SERVICE =
  new InjectionToken<StorageService>('STORAGE_SERVICE');
```

用途：

> 提供 Angular Runtime 一個 DI 可以辨識的「鑰匙」。

它在 Runtime 中存在。

因此可以理解成：

```text
StorageService
    ↓
Interface
    ↓
「規格」

STORAGE_SERVICE
    ↓
InjectionToken
    ↓
「Angular DI 的鑰匙」
```

---

# 13. 第五步：在 Provider 指定實作

例如正式環境希望使用：

```text
LocalStorageService
```

可以：

```typescript
providers: [
  {
    provide: STORAGE_SERVICE,
    useClass: LocalStorageService
  }
]
```

意思是：

```text
STORAGE_SERVICE
       ↓
Angular DI 找到這個 Token
       ↓
使用 LocalStorageService
```

---

# 14. Component 如何注入？

```typescript
import { Component, Inject } from '@angular/core';
import { STORAGE_SERVICE } from './storage.token';
import { StorageService } from './storage.service.interface';

@Component({
  selector: 'app-user',
  templateUrl: './user.component.html',
  providers: [
    {
      provide: STORAGE_SERVICE,
      useClass: LocalStorageService
    }
  ]
})
export class UserComponent {

  constructor(
    @Inject(STORAGE_SERVICE)
    private storageService: StorageService
  ) {}

  saveUser(): void {
    this.storageService.set('username', 'WeiHao');
  }

  getUser(): string | null {
    return this.storageService.get('username');
  }

}
```

這裡其實同時使用了兩種不同的東西。

### `StorageService`

```typescript
private storageService: StorageService
```

是：

> TypeScript 型別。

### `STORAGE_SERVICE`

```typescript
@Inject(STORAGE_SERVICE)
```

是：

> Angular DI Runtime 使用的 Token。

---

# 15. 完整流程

整個關係可以畫成：

```text
StorageService
      │
      │ Interface
      │ 定義規格
      ↓
STORAGE_SERVICE
      │
      │ InjectionToken
      │ Angular Runtime 的 DI Token
      ↓
Provider
      │
      │ provide: STORAGE_SERVICE
      │ useClass: LocalStorageService
      ↓
LocalStorageService
      │
      ↓
Angular DI Container
      │
      ↓
UserComponent
```

因此：

```typescript
constructor(
  @Inject(STORAGE_SERVICE)
  private storageService: StorageService
)
```

實際拿到的是：

```text
LocalStorageService instance
```

而不是 Interface。

Interface 只負責定義型別契約。

---

# 16. 如果要換成 SessionStorageService

只需要改 Provider：

```typescript
providers: [
  {
    provide: STORAGE_SERVICE,
    useClass: SessionStorageService
  }
]
```

Component 完全不需要修改。

原本：

```text
UserComponent
      ↓
STORAGE_SERVICE
      ↓
LocalStorageService
```

改成：

```text
UserComponent
      ↓
STORAGE_SERVICE
      ↓
SessionStorageService
```

這就是 Interface + DI 的實際價值。

---

# 17. Unit Test 如何替換 Service？

正式環境：

```text
STORAGE_SERVICE
       ↓
LocalStorageService
```

Unit Test 可能不希望真的操作 Browser LocalStorage。

可以建立 Mock：

```typescript
const mockStorageService: StorageService = {
  get: jasmine.createSpy('get'),
  set: jasmine.createSpy('set')
};
```

然後：

```typescript
TestBed.configureTestingModule({
  providers: [
    {
      provide: STORAGE_SERVICE,
      useValue: mockStorageService
    }
  ]
});
```

測試環境變成：

```text
STORAGE_SERVICE
       ↓
mockStorageService
       ↓
Jasmine Spy
```

Component 不需要修改。

這就是 DI 在 Unit Test 中非常重要的用途：

> **讓測試環境可以替換正式環境的依賴。**

---

# 18. `useClass`、`useValue`、`useFactory`

Angular Provider 常見的幾種方式：

## `useClass`

```typescript
{
  provide: STORAGE_SERVICE,
  useClass: LocalStorageService
}
```

表示：

> 使用指定的 Class 建立實例。

---

## `useValue`

```typescript
{
  provide: STORAGE_SERVICE,
  useValue: mockStorageService
}
```

表示：

> 直接提供一個已經存在的物件。

Unit Test 很常使用。

---

## `useFactory`

```typescript
{
  provide: STORAGE_SERVICE,
  useFactory: () => {
    return new LocalStorageService();
  }
}
```

表示：

> 透過 Factory 建立要注入的物件。

適合建立邏輯比較複雜的情況。

---

# 19. 這些東西一定要全部寫在同一個 `.ts` 嗎？

不需要。

技術上可以全部寫在同一個檔案，但正式專案通常會按照職責拆開。

例如：

```text
services/
│
├── storage.service.interface.ts
├── storage.token.ts
├── local-storage.service.ts
└── session-storage.service.ts

components/
│
└── user.component.ts
```

各檔案的責任：

```text
storage.service.interface.ts
        ↓
定義 Service 規格

storage.token.ts
        ↓
定義 Angular DI Token

local-storage.service.ts
        ↓
LocalStorage 實作

session-storage.service.ts
        ↓
SessionStorage 實作

user.component.ts
        ↓
使用 StorageService
```

這樣比全部塞在 Component 裡容易維護。

---

# 20. 為什麼 `storage.token.ts` 可以單獨存在？

因為 Token 本身就是一個獨立的 DI 設定概念。

例如：

```typescript
export const STORAGE_SERVICE =
  new InjectionToken<StorageService>('STORAGE_SERVICE');
```

它不需要知道：

- LocalStorage 怎麼實作
- SessionStorage 怎麼實作
- Component 怎麼使用
- Unit Test 怎麼 Mock

它只負責：

```text
「StorageService 這個依賴，在 Angular Runtime 中叫做什麼 Token？」
```

真正的實作由 Provider 決定。

因此可以有：

```text
storage.token.ts
        ↓
STORAGE_SERVICE
        ↓
┌───────────────┬────────────────┐
↓               ↓                ↓
LocalStorage    SessionStorage   Mock
```

同一個 Token 可以在不同環境對應不同實作。

---

# 21. Angular Interface 更常見的用途

雖然 Angular 可以使用 Interface + DI，但實務上 Angular 的 Interface 更常拿來描述「資料」。

例如：

```typescript
export interface User {
  id: number;
  name: string;
  email: string;
}
```

Service：

```typescript
getUser(): Observable<User> {
  return this.http.get<User>('/api/user');
}
```

這時 Interface 的作用是：

> 告訴 TypeScript `User` 這筆資料應該長什麼樣子。

例如：

```text
User
├── id: number
├── name: string
└── email: string
```

---

# 22. Angular 常見的 Interface

例如：

```text
User
Product
ApiResponse
LoginRequest
LoginResponse
AppConfig
MenuItem
TableColumn
FormData
```

例如：

```typescript
export interface LoginRequest {
  username: string;
  password: string;
}

export interface LoginResponse {
  token: string;
  expires: number;
}
```

這種用法在 Angular / TypeScript 中非常常見。

---

# 23. 為什麼 Angular Interface 沒有 .NET Core 那麼常拿來做 Service？

主要是因為 TypeScript Interface 的特性不同。

.NET Core：

```text
IUserService
      ↓
DI
      ↓
UserService
```

Interface 很常用於：

- Service 抽象
- DI
- 替換實作
- Unit Test Mock

Angular / TypeScript：

```text
User
Product
ApiResponse
LoginRequest
```

Interface 更常用於：

- 描述 API 資料
- 描述 Component 資料
- 描述表單資料
- 描述設定物件
- 提供 TypeScript 型別檢查

所以：

> **不要因為 .NET Core 很常使用 Interface + DI，就認為 Angular 每一個 Service 都應該建立 Interface。**

---

# 24. 什麼時候值得使用 Interface + InjectionToken？

比較適合：

### ① 同一個契約有多個實作

例如：

```text
StorageService
├── LocalStorageService
├── SessionStorageService
└── MemoryStorageService
```

需要根據不同環境切換實作。

### ② 希望 Component 與底層實作解耦

Component 只知道：

```text
StorageService
```

不需要知道底層是：

```text
LocalStorage
SessionStorage
IndexedDB
Memory
```

### ③ Unit Test 需要替換依賴

正式環境：

```text
STORAGE_SERVICE
      ↓
LocalStorageService
```

測試環境：

```text
STORAGE_SERVICE
      ↓
MockStorageService
```

---

# 25. 什麼時候不需要 Interface + InjectionToken？

如果只有單一 Service：

```typescript
@Injectable({
  providedIn: 'root'
})
export class UserService {
}
```

而且：

- 沒有多個實作
- 不需要特別抽象化
- 直接使用 Class 就很清楚

那麼直接：

```typescript
constructor(
  private userService: UserService
) {}
```

通常就足夠。

不需要為了模仿 .NET Core 而額外建立：

```text
IUserService
USER_SERVICE
UserService
```

三層結構。

---

# 26. Angular DI 與 .NET Core DI 對照

| | .NET Core | Angular |
|---|---|---|
| DI Container | .NET DI | Angular DI |
| 常見 DI Token | Interface / Class | Class / InjectionToken |
| Interface | 常用於 Service 抽象 | 常用於資料結構 |
| Interface Runtime 存在 | 可以作為 Runtime Type | TypeScript Interface 會消失 |
| 替換實作 | `IService → Implementation` | `provide → useClass/useValue/useFactory` |
| Unit Test | DI 替換 Implementation | TestBed Provider 替換實例 |
| 一般 Service | Interface + Class 很常見 | 直接使用 Class 很常見 |

---

# 27. 最重要的四個觀念

### ① Angular 也有 DI

```text
Component
   ↓
Angular DI
   ↓
Service
```

### ② TypeScript Interface 不能直接當 Angular DI Token

因為：

```typescript
interface IUserService {}
```

編譯成 JavaScript 後會消失。

---

### ③ Interface 與 InjectionToken 是不同東西

```text
StorageService
    ↓
Interface
    ↓
定義「規格」
```

```text
STORAGE_SERVICE
    ↓
InjectionToken
    ↓
Angular Runtime 的 DI「鑰匙」
```

---

### ④ Provider 決定實際使用哪個實作

例如：

```typescript
{
  provide: STORAGE_SERVICE,
  useClass: LocalStorageService
}
```

代表：

> 當 Angular 發現有人要求 `STORAGE_SERVICE` 時，就提供 `LocalStorageService`。

---

# 28. 一句話總結

> **.NET Core 常使用「Interface → Implementation → DI」建立 Service 抽象；Angular 也可以做到，但因為 TypeScript Interface 在 Runtime 會消失，所以需要透過 `InjectionToken` 作為 Angular DI 的識別依據。實務上 Angular 的 Interface 更多是拿來定義資料結構，而不是每個 Service 都建立一個 Interface。**

---

# 29. 最簡單的記憶方式

可以把整個 Angular Interface + DI 想成：

```text
Interface
    ↓
「規格是什麼？」

InjectionToken
    ↓
「Angular DI 要找哪個依賴？」

Provider
    ↓
「這個依賴實際使用哪個實作？」

Component
    ↓
「我只管使用，不管實際是哪個實作。」
```

也就是：

```text
StorageService
      ↓
規格

STORAGE_SERVICE
      ↓
DI Token

provide + useClass / useValue
      ↓
實際實作

UserComponent
      ↓
使用依賴
```

