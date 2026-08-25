---
title: Angular 與 .NET Core 的依賴注入(DI)比較筆記_Claude
date: 2026-08-24 16:09:50
categories:
 - 前端開發
tags:
 - Angular
 - AI產出
---
<!-- # Angular 與 .NET Core 的依賴注入(DI)比較筆記 -->

從 .NET Core 轉來寫 Angular 時,最容易卡住的地方就是依賴注入(Dependency Injection, DI)的設定方式。這篇整理幾個常見疑問,把兩邊的機制做個對照。
<span style="color:red;">文章內容透過ai整理產出</span>
<!--more-->
---

## .NET Core:DI 容器設定在哪裡?

要看版本:

- **.NET 6 以後**(minimal hosting model,目前預設模板):直接寫在 `Program.cs`,用 `builder.Services` 註冊服務。

  ```csharp
  var builder = WebApplication.CreateBuilder(args);

  builder.Services.AddControllers();
  builder.Services.AddScoped<IMyService, MyService>();

  var app = builder.Build();
  ```

- **.NET Core 3.1 / .NET 5**(舊版模板):寫在 `Startup.cs` 的 `ConfigureServices(IServiceCollection services)`,`Program.cs` 只負責呼叫 `CreateHostBuilder` 啟動,不會直接看到服務註冊。

## Angular:DI 設定在哪裡?

Angular 的設定位置比較分散,依範圍大小分幾層:

| 範圍 | 設定位置 |
|---|---|
| 全域(standalone 架構,Angular 14+ / 17 之後預設) | `app.config.ts` 的 `ApplicationConfig.providers` |
| 全域(傳統 NgModule 架構) | `app.module.ts` 的 `@NgModule({ providers: [...] })` |
| 服務自身宣告全域單例 | `@Injectable({ providedIn: 'root' })` |
| 元件子樹範圍 | `@Component({ providers: [...] })` |

範例(standalone):

```typescript
export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    { provide: MyService, useClass: MyServiceImpl }
  ]
};
```

## 為什麼沒在 `app.config.ts` 設定,直接在 constructor 注入也能用?

```typescript
constructor(private _layerService: LayerService) {}
```

如果這樣寫也能正常運作,代表 `LayerService` 這個 class 上面本身就有:

```typescript
@Injectable({
  providedIn: 'root'
})
export class LayerService { ... }
```

這是 Angular 的 **tree-shakable providers** 機制:服務只要在自己的 `@Injectable` decorator 上宣告 `providedIn: 'root'`,Angular 就會自動把它註冊到 root injector,不需要額外寫進任何 `providers` 陣列。好處是如果這個服務完全沒被用到,打包時會被 tree-shaking 移除。

反過來說,如果服務沒有 `providedIn`,又沒有在任何地方手動註冊,直接注入會在執行時拋出:

```
NullInjectorError: No provider for LayerService!
```

## 如何替換掉實例?

DI 的核心是「token 對應到實際提供的東西」,替換就是改變這個對應關係。常見寫法:

```typescript
// 換成另一個 class 實作
{ provide: MyService, useClass: MockMyService }

// 直接給一個現成物件/值
{ provide: MyService, useValue: { getData: () => 'fake data' } }

// 用工廠函式動態決定
{
  provide: MyService,
  useFactory: (http: HttpClient) => environment.production
    ? new RealService(http)
    : new FakeService(),
  deps: [HttpClient]
}

// 別名到另一個已存在的 token
{ provide: OldService, useExisting: NewService }
```

設定放的位置決定「覆蓋範圍」:放在 `app.config.ts` / root module 是全域替換;放在某個元件的 `providers` 只影響該元件子樹;測試時則建議放在 `TestBed.configureTestingModule` 裡,避免動到正式程式碼。

## 集中管理替換設定,比逐個修改 Service 檔案好嗎?

是的,通常建議把「該用哪個實作」的決策集中在 `app.config.ts`(或啟動設定),而不是每次去改 service 檔案本身,理由:

- **單一真相來源**:所有環境/情境的替換決策都在一個地方,方便追蹤與 code review。
- **service 保持乾淨**:service 本身不該知道「誰會替換它」,這種決策屬於組裝(composition)層級,符合依賴反轉原則。

實務作法是先定義抽象契約(見下一節),再依環境決定實作:

```typescript
export abstract class LayerService {
  abstract getLayers(): Layer[];
}

@Injectable()
export class RealLayerService extends LayerService { ... }

@Injectable()
export class MockLayerService extends LayerService { ... }
```

```typescript
export const appConfig: ApplicationConfig = {
  providers: [
    { provide: LayerService, useClass: environment.useMock ? MockLayerService : RealLayerService }
  ]
};
```

例外:如果只是某個元件子樹需要不同實作,放在該元件的 `providers` 反而更清楚,因為影響範圍本來就侷限在那裡。

## 為什麼 .NET Core 用 interface 當 DI 契約,Angular 卻不能直接用 interface?

關鍵差異在於「執行時期是否存在」:

- **C# 的 interface**:編譯成 IL 後仍是 CLR 型別系統裡真實存在的型別,可以用 reflection 或 `typeof(IMyService)` 當 key,所以 DI 容器能拿它做查表。
- **TypeScript 的 interface**:純粹是編譯期的型別檢查工具,轉譯成 JavaScript 後會被完全抹除,執行時期根本不存在這個東西。

如果寫:

```typescript
{ provide: ILayerService, useClass: LayerServiceImpl } // ❌ 編譯錯誤
```

TypeScript 會直接報錯:`'ILayerService' only refers to a type, but is being used as a value here`,因為 DI 容器在執行時期需要一個「真實存在的值」當 token,interface 不符合這個要求。

Angular 的替代方案是找「編譯期是型別、執行期也真實存在的東西」:

1. **abstract class**:編譯後仍是真實的 JS function/物件,可以同時當 token 又當型別。
2. **InjectionToken** 搭配 interface 只做編譯期型別檢查(見下一節)。

## Angular 的 interface 實際用在哪裡?

TypeScript 是結構型別系統(structural typing),不是像 C# 的名義型別系統(nominal typing),天生比較適合描述「資料形狀」而不是「可替換的行為」。常見用途:

1. **定義資料模型**(最常見)

   ```typescript
   export interface Layer {
     id: string;
     name: string;
     visible: boolean;
   }
   ```

2. **元件的 `@Input` / `@Output` 型別**

   ```typescript
   @Input() layer: Layer;
   ```

3. **實作 Angular 生命週期介面**(接近行為契約的用法)

   ```typescript
   export class MyComponent implements OnInit, OnDestroy {
     ngOnInit(): void { ... }
     ngOnDestroy(): void { ... }
   }
   ```

   注意:Angular 執行時期不是靠檢查 `implements OnInit` 來呼叫 `ngOnInit`,而是單純檢查 class 上有沒有這個名字的方法(慣例呼叫)。`implements OnInit` 純粹是給開發者在編譯期防呆用。

4. **function 參數/回傳值型別、泛型限制**

   ```typescript
   function renderLayers<T extends Layer>(layers: T[]): void { ... }
   ```

5. **描述設定/選項物件**

   ```typescript
   interface HttpOptions {
     headers?: Record<string, string>;
     timeout?: number;
   }
   ```

## Angular 也能做到「Interface + DI」:完整範例

只要搭配 `InjectionToken`,Angular 也能做到跟 C# `IMyService` 注入一樣的解耦效果。實務上會拆成多個檔案,不會全部塞在一起:

**`layer.model.ts`(資料模型)**

```typescript
export interface Layer {
  id: string;
  name: string;
  visible: boolean;
}
```

**`layer.service.interface.ts`(契約 + token,習慣放同一個檔案)**

```typescript
import { InjectionToken } from '@angular/core';
import { Layer } from './layer.model';

export interface ILayerService {
  getLayers(): Layer[];
  addLayer(layer: Layer): void;
}

export const LAYER_SERVICE = new InjectionToken<ILayerService>('LayerService');
```

**`real-layer.service.ts`(正式實作)**

```typescript
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { ILayerService } from './layer.service.interface';
import { Layer } from './layer.model';

@Injectable()
export class RealLayerService implements ILayerService {
  constructor(private http: HttpClient) {}

  getLayers(): Layer[] {
    return [];
  }

  addLayer(layer: Layer): void { }
}
```

**`mock-layer.service.ts`(測試/假資料實作)**

```typescript
import { Injectable } from '@angular/core';
import { ILayerService } from './layer.service.interface';
import { Layer } from './layer.model';

@Injectable()
export class MockLayerService implements ILayerService {
  getLayers(): Layer[] {
    return [{ id: '1', name: 'Fake Layer', visible: true }];
  }

  addLayer(layer: Layer): void {
    console.log('mock add', layer);
  }
}
```

**`app.config.ts`(組裝,決定注入哪個實作)**

```typescript
import { ApplicationConfig } from '@angular/core';
import { LAYER_SERVICE } from './layer.service.interface';
import { RealLayerService } from './real-layer.service';
import { MockLayerService } from './mock-layer.service';
import { environment } from '../environments/environment';

export const appConfig: ApplicationConfig = {
  providers: [
    {
      provide: LAYER_SERVICE,
      useClass: environment.useMock ? MockLayerService : RealLayerService
    }
  ]
};
```

**元件內注入使用**

```typescript
import { Component, Inject } from '@angular/core';
import { LAYER_SERVICE, ILayerService } from './layer.service.interface';

@Component({ ... })
export class MapComponent {
  constructor(@Inject(LAYER_SERVICE) private layerService: ILayerService) {}

  ngOnInit() {
    const layers = this.layerService.getLayers();
  }
}
```

這裡 `layerService` 的型別是 `ILayerService`(interface),元件只能呼叫 interface 定義的方法,也完全不需要 import 任何具體 class(`RealLayerService` / `MockLayerService`),真正做到跟實作解耦——這是比 abstract class 更貼近 C# interface 精神的寫法,因為 abstract class 畢竟還是一個 class,元件多少會 import 到它;用 interface + token 時,元件只 import 一個純型別的 interface(編譯後消失)跟一個 token。

## 那實務上為什麼還是以 class 直接注入為主,很少真的用 Interface + DI?

雖然上面證明 Angular 做得到,但實務上大多數 Angular 專案很少這樣寫,主流仍是直接注入具體的 class。原因如下:

**1. 具體 class 本身就已經是「可替換的 token」了**

這是最核心的一點。C# 需要 interface 才能做到「注入時可替換」,是因為 C# 對一個具體 class 的依賴通常代表緊耦合、甚至有 sealed 的限制。但在 Angular/TypeScript 裡,一個具體的 class(例如 `RealLayerService`)本身在執行時期就是真實存在的值,完全可以直接拿來當 provider token:

```typescript
{ provide: RealLayerService, useClass: MockLayerService }
```

只要 `MockLayerService` extends 或至少結構相容 `RealLayerService`,一樣可以替換成功。也就是說,多加一層 `interface + InjectionToken` 並沒有換來額外的替換能力——具體 class 已經免費具備這個能力了,額外那層只是多維護一份程式碼。

**2. 測試時不需要靠 interface 就能 mock**

在 .NET 裡,mock 一個 concrete class 有時會有限制(沒有 virtual 方法就 mock 不了),所以養成「一定要 interface」的習慣。但在 TypeScript 生態,`jasmine.createSpyObj`、Jest 的 `jest.mock()`,或 Angular 的：

```typescript
TestBed.overrideProvider(RealLayerService, { useValue: mockObj });
```

都可以直接針對具體 class 做替換或建立假物件,不需要先抽出 interface 才能測試。

**3. 少了 `@Inject()` 的額外語法負擔**

用具體 class 當 token,注入語法最乾淨:

```typescript
constructor(private layerService: LayerService) {}
```

用 `InjectionToken` 則一定要加 `@Inject()`:

```typescript
constructor(@Inject(LAYER_SERVICE) private layerService: ILayerService) {}
```

多一道語法,團隊協作時大家很自然會選省事的那條路。

**4. `providedIn: 'root'` 這種 tree-shakable 寫法只有 class 才能享有**

`InjectionToken` 沒有這種零設定的便利性,一定要手動在某個 `providers` 裡註冊,等於多了一道維護成本。

**5. YAGNI——大多數服務一輩子只有一種實作**

企業級 .NET 後端常見 Repository Pattern、多資料庫切換、分層架構,「多實作」是常態需求。但前端的 Angular service 大多數情況下(呼叫某個 API、管理某塊 UI 狀態)這輩子就只會有一種實作,不會有第二個版本要替換。既然沒有「未來會替換」的實際需求,先做 interface + token 這層抽象就是過度設計,平白增加檔案數跟認知負擔。

**什麼時候才真的值得用 Interface + DI?**

值得用的情境反而比較單一:寫給第三方用的 library / SDK,不想暴露具體實作;或明確知道未來一定要做多環境切換(例如瀏覽器 storage vs. server-side storage 這種平台差異)。Angular 官方自己的套件(Material、CDK)在需要「使用者可自訂實作」的地方,也確實會用 `InjectionToken`,但那是刻意設計給外部消費者擴充用的,不是預設的服務注入模式。

簡單說:Angular 選擇「直接注入具體 class」不是因為做不到 interface + DI,而是因為在 TypeScript 的執行環境下,這層抽象幾乎不會多換來任何實際好處,卻要多付語法跟維護成本——這跟 C# 的情境剛好相反。

## 判斷標準是「abstract class」還是「有沒有明確的替換點」?

一個常見的誤解是:「Service 可能有多個實例就做成 abstract class 方便切換,只有一個實例就用預設的 `providedIn: 'root'`」。這個方向大致正確,但有兩個細節需要修正。

**「能不能切換」不是 abstract class 才有的特權**

一個具體 class 就算沒有做成 abstract,一樣可以在 provider 設定裡被替換:

```typescript
{ provide: RealLayerService, useClass: MockLayerService }
```

所以嚴格來說,「能不能切換實例」這件事,concrete class 本身就具備,不是 abstract class 才有的特權。

**真正該用「是否只有一種實作」判斷的,其實是這樣:**

- **只會有一種實作,而且不預期要替換** → 用具體 class + `providedIn: 'root'`,零設定、最省事。
- **確定會有多種實作**(測試用 mock、多環境切換、給第三方擴充)→ 做成 abstract class(或 `InjectionToken`)有意義,但重點不是「因此才能替換」,而是:
  1. **強制所有消費者只依賴共同契約**,不會有人不小心呼叫到某個具體實作獨有、契約外的方法,導致換掉實作時某處爆炸。
  2. **文件效果**:一看到 `abstract class LayerService`,就知道這個 token 天生設計成會有多個實作,不用另外寫註解說明。

**abstract class 沒有 `providedIn: 'root'` 這個選項可用**

因為 abstract class 不能被實例化,Angular 沒辦法「預設自動建立它自己」當 provider,所以一旦做成 abstract class,就**必須**在某處(`app.config.ts` 或元件 `providers`)明確寫 `useClass` 指定要用哪個具體實作,否則會拿到 `NullInjectorError`。這反而是個天然的強制機制:一旦選擇 abstract class,就等於承諾「這裡一定會有人手動決定用哪個實作」。

**修正後的判斷準則:**

> 只會有一種實作、不預期替換 → 具體 class + `providedIn: 'root'`,省事。
> 預期會有多種實作、需要明確要求所有消費者只依賴共同契約 → 做成 abstract class(或 InjectionToken),並在組裝層明確指定用哪個實作。

**abstract class 跟 InjectionToken 怎麼選?**

如果多個實作之間本來就適合用 OOP 繼承(`extends`)表達,abstract class 比較自然;如果注入的東西不是 class 形式(設定值、字串、原始型別),或不想用繼承綁死結構,`InjectionToken` 比較合適。

## 小結:兩邊的心智模型對照

| 概念 | .NET Core | Angular |
|---|---|---|
| 服務註冊位置 | `Program.cs`(新版)/ `Startup.cs`(舊版) | `app.config.ts` / `NgModule` / `@Injectable({providedIn:'root'})` / 元件 `providers` |
| 契約型別 | interface(執行時期真實存在) | interface 只在編譯期存在,需搭配 abstract class 或 `InjectionToken` 才能當 DI token |
| 替換實作 | `AddScoped<IMyService, Impl>()` | `{ provide: Token, useClass/useValue/useFactory/useExisting: ... }` |
| interface 的定位 | 行為契約 + 執行時期多型 | 主要是資料形狀的編譯期規格書 |