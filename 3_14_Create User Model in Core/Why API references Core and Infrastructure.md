# 為什麼 API 同時參照 Core 與 Infrastructure?

> **一句話**:為了在 `Program.cs` 登記服務(DI)而必須參照,但**只准在 `Program.cs` 用**,Controller 絕不直接碰 Infrastructure。

## 兩個必須參照的原因

| 原因 | 說明 |
|---|---|
| `Program.cs` 要做 DI 登記 | `AddInfrastructure()` 在 Infrastructure、`AddCore()` 在 Core,不參照就編譯不過 |
| API 是「接線生」 | Controller 接到請求 → 呼叫 Core 的 Service → 回傳結果,所以一定要參照 Core |

## 這不是破壞架構嗎?

技術上 API 參照了 Infrastructure,Controller 可以繞過 Core 直接用 `DbContext`。實務上靠**團隊約定**禁止:

> API 對 Infrastructure 的使用**僅限** `Program.cs` 的 `builder.Services.AddInfrastructure()`。
> 寫 Controller 時,只准跟 Core 講話。

這是 Clean Architecture 在實務上的「技術妥協」,編譯器不會幫你擋,要靠自律與 code review。

## 類比:餐廳

| 角色 | 對應 |
|---|---|
| Core | 總主廚(研發菜單、掌控烹飪) |
| Infrastructure | 烤箱、冰箱、物流設備 |
| API(`Program.cs`) | 老闆:開店第一天引進主廚與設備,之後退到幕後 |

客人點餐時,由主廚發號施令、主廚去用烤箱,老闆不插手烹飪細節。
