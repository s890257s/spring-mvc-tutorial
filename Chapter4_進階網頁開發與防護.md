# 章節 4 ｜ 進階網頁開發實戰：檔案操作、分頁查詢與資料防護

---

## <a id="toc"></a>目錄

- [4-1 進階資料檢索：分頁、排序與動態查詢](#CH4-1)
  - [1. 結合 Pageable 介面實現分頁與自訂排序](#CH4-1-1)
  - [2. 前端 Thymeleaf 的進階分頁按鈕邏輯渲染](#CH4-1-2)
  - [3. 動態查詢實戰：運用 Specification 動態拼裝查詢條件](#CH4-1-3)
- [4-2 資料驗證：Spring Validation](#CH4-2)
  - [1. 實作 DTO 欄位驗證 (@NotBlank, @Size, @Pattern)](#CH4-2-1)
  - [2. Controller 攔截 @Valid 與 BindingResult 錯誤解析](#CH4-2-2)
  - [3. Thymeleaf 表單錯誤訊息的動態回顯 (Error Rendering)](#CH4-2-3)
- [4-3 系統日誌與全域例外處理](#CH4-3)
  - [1. 告別 System.out：使用 @Slf4j 與 Logback 紀錄層級日誌](#CH4-3-1)
  - [2. 認識 @ControllerAdvice 與 @ExceptionHandler](#CH4-3-2)
  - [3. 實作：錯誤頁面模板與自訂例外類別](#CH4-3-3)

---

## <a id="CH4-1"></a>[4-1 進階資料檢索：分頁、排序與動態查詢](#toc)

### <a id="CH4-1-1"></a>[1. 結合 Pageable 介面實現分頁與自訂排序](#toc)

**📍 單元目標**  
藉由 Spring Data JPA 內建的分頁機制，輕鬆實現資料庫抓取筆數限制與資料排序功能。學完本單元後，你將能夠在任何 Repository 上掛載分頁查詢，並靈活運用多欄位排序組合。

**🤔 為什麼需要它**

想像一下，你經營的電商網站已經累積了 50 萬筆商品資料。某天，使用者點進「全部商品」頁面，後端執行了一句：

```sql
SELECT * FROM product;
```

這條 SQL 會一口氣把 50 萬筆資料從資料庫拉進 Java 的記憶體裡，光是網路傳輸就會耗費好幾秒鐘，也因 Java 會為每一筆資料建立物件實例，記憶體使用量可能瞬間暴增至數 GB，最後還要把這一大坨資料序列化成 HTML 再傳給瀏覽器。結果就是：使用者等了 30 秒只看到白畫面，伺服器也可能因記憶體不足而直接崩潰。

在真實世界中，人的眼球一個畫面最多只能消化 10～20 筆資料。所以解法很直覺：**一次只查「當頁需要」的那幾筆就好**。這就是分頁 Pagination 的核心精神。

在 SQL 層級，分頁查詢靠的是 `LIMIT` 與 `OFFSET` 兩個關鍵字：

```sql
-- 抓取第 3 頁的資料，每頁 10 筆
-- OFFSET 20 代表跳過前 20 筆，LIMIT 10 代表只抓接下來的 10 筆
SELECT * FROM product ORDER BY price DESC LIMIT 10 OFFSET 20;
```

不過，如果每次都要自己手動計算 `OFFSET` 值並拼組 SQL，不但容易出錯，程式碼也難以複用。Spring Data JPA 透過 `Pageable` 介面幫我們把這一整套「算頁碼 → 組 SQL → 包裝結果」的流程全面自動化。

**📖 核心概念**

Spring Data JPA 的分頁機制由以下幾個核心元件協同運作：

| 核心元件               | 角色定位     | 運作說明                                                                                                                  | 使用範例                                                                                                                                                                                                                   |
| :--------------------- | :----------- | :------------------------------------------------------------------------------------------------------------------------ | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **`Pageable`**         | 介面規格     | 定義分頁請求該具備的資訊，包含頁碼、每頁筆數與排序規則。它是一個介面，我們通常不直接實作它，而是透過 `PageRequest` 建立。 | `Pageable p = PageRequest.of(0, 10);`                                                                                                                                                                                      |
| **`PageRequest.of()`** | 工廠方法     | `Pageable` 最常用的實作方式。快速產出包含當前頁數、每頁筆數以及排序規則的請求物件。                                       | `PageRequest.of(2, 20, Sort.by("price").descending())`<br>第 3 頁、每頁 20 筆、價格降冪                                                                                                                                    |
| **`Sort`**             | 排序規則     | 獨立的排序定義物件，可指定針對哪些欄位排序以及升冪 ASC 或降冪 DESC。也可獨立於分頁使用。                                  | `Sort.by("price").descending()` <br>產生 `ORDER BY price DESC`，也就是對 `price` 進行降冪排序<br><br> `Sort.by("id").ascending()`<br>`.and(Sort.by("name").descending());`<br>先對 `id` 做升冪排序，再對 `name` 做降冪排序 |
| **`Page<T>`**          | 完整分頁結果 | 執行回傳的分頁包裹物件，內部除了包含當頁的 List 資料外，還自帶總頁數、總筆數、是否有上下頁等完整資訊。                    | `Page<Product> result = repository.findAll(pageable);` 回傳含當頁資料 + 總頁數等資訊                                                                                                                                       |
| **`Slice<T>`**         | 輕量分頁結果 | 與 `Page<T>` 類似，但**不會執行 `COUNT` 計算總筆數**，適合「載入更多」的無限捲動情境，效能更佳。                          | `Slice<Product> result = repository.findByName(pageable);` 回傳含當頁資料 + 是否有下一頁<br>(需於介面中定義)                                                                                                               |

`Page<T>` 物件內建了非常豐富的輔助方法，在渲染前端分頁按鈕時極為好用：

假設資料庫共有 **95 筆**商品，我們請求 `PageRequest.of(2, 10)` 即第 3 頁、每頁 10 筆：

| 常用方法                | 回傳型別  | 說明                                               | 本例回傳值                             |
| :---------------------- | :-------- | :------------------------------------------------- | :------------------------------------- |
| `getContent()`          | `List<T>` | 取得當頁的資料合集。                               | 第 21～30 筆商品的 List                |
| `getTotalPages()`       | `int`     | 取得總頁數。                                       | `10`（95 ÷ 10 無條件進位）             |
| `getTotalElements()`    | `long`    | 取得資料庫中符合條件的總資料筆數。                 | `95`                                   |
| `getNumber()`           | `int`     | 取得目前頁碼，注意是底層從 0 起算的值。            | `2`（對使用者來說是第 3 頁）           |
| `getSize()`             | `int`     | 取得每頁設定的筆數上限。                           | `10`                                   |
| `getNumberOfElements()` | `int`     | 取得當頁**實際回傳**的筆數，最後一頁可能不足滿頁。 | `10`（若查最後一頁 page=9 則回傳 `5`） |
| `hasNext()`             | `boolean` | 是否還有下一頁。                                   | `true`（後面還有第 4～10 頁）          |
| `hasPrevious()`         | `boolean` | 是否還有上一頁。                                   | `true`（前面有第 1、2 頁）             |
| `isFirst()`             | `boolean` | 是否為第一頁。                                     | `false`（第一頁是 page=0）             |
| `isLast()`              | `boolean` | 是否為最後一頁。                                   | `false`（最後一頁是 page=9）           |

> 💡 **知識補充**
> `Page<T>` 底層會額外執行一條 `SELECT COUNT(*) FROM ...` 來計算總筆數，這在資料量極大的情境下可能成為效能瓶頸。如果你的前端是「載入更多」的無限捲動設計，而非傳統翻頁按鈕，可以改用 `Slice<T>`，它只會多查一筆來判斷是否有下一頁，省去了 `COUNT` 的開銷。

**💻 實作範例**

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;

@Controller
public class ProductPageController {

    @Autowired
    private ProductRepository productRepository;

    @GetMapping("/products/page")
    public String viewProductPage(
            // 接收前端要求查看的頁數，預設第 1 頁
            @RequestParam(defaultValue = "1") Integer pageNumber,
            Model model) {

        // 💡 JPA 分頁機制是從 0 開始計算的，所以傳入的 pageNumber 必須減 1
        int jpaPage = pageNumber - 1;
        int pageSize = 10; // 每頁顯示 10 筆

        // 建立分頁請求規則，並設定依照價格 price 進行降冪排序，從高到低排
        Pageable pageable = PageRequest.of(jpaPage, pageSize, Sort.by("price").descending());

        // 使用 JPA 內建的 findAll 方法，並傳入 Pageable 設定
        Page<Product> pageResult = productRepository.findAll(pageable);

        // 將結果裝箱送往前台
        model.addAttribute("page", pageResult);
        model.addAttribute("currentPage", pageNumber); // 傳送當前實際頁碼供前端渲染按鈕

        return "product-page-view";
    }
}
```

**🔍 逐行拆解**

1. `@RequestParam(defaultValue = "1") Integer pageNumber`  
   從 URL 的 Query String 接收前端傳來的頁碼參數。例如使用者點了第 3 頁按鈕，瀏覽器會發出 `/products/page?pageNumber=3`。若未傳入則預設為 1。

2. `int jpaPage = pageNumber - 1;`  
   **關鍵換算**。使用者認知的「第 1 頁」等於 JPA 底層的「第 0 頁」。如果忘記減 1，使用者看到的第 1 頁實際上會顯示第 2 頁的資料，造成資料錯位。

3. `PageRequest.of(jpaPage, pageSize, Sort.by("price").descending())`  
   建立一個分頁請求物件。這段的意思是：請幫我查詢第 `jpaPage` 頁、每頁 `pageSize` 筆資料，並且依照 `price` 欄位由大到小降冪排列。JPA 會自動將此轉譯為 SQL 的 `ORDER BY price DESC LIMIT 10 OFFSET ?`。

4. `productRepository.findAll(pageable)`  
   呼叫 JPA 內建的 `findAll` 方法並傳入分頁規則。`JpaRepository` 已預先宣告 `Page<T> findAll(Pageable pageable)` 這個方法簽章，因此我們不需要額外撰寫任何 SQL。

5. `model.addAttribute("page", pageResult)`  
   將整個 `Page` 物件放入 Model，前端 Thymeleaf 就能透過 `${page.content}` 取得當頁資料集合，也能呼叫 `${page.totalPages}` 取得總頁數等資訊。

**📐 `PageRequest.of()` 的三種多載寫法**

依據你的需求，`PageRequest.of()` 提供了多種建構方式：

| 多載方法簽章                                                          | 用途說明                       | 使用情境                     |
| :-------------------------------------------------------------------- | :----------------------------- | :--------------------------- |
| `PageRequest.of(int page, int size)`                                  | 只指定頁碼與筆數，不排序       | 不在意資料排列順序時使用     |
| `PageRequest.of(int page, int size, Sort sort)`                       | 頁碼、筆數加上排序規則         | 最常用，搭配獨立的 Sort 物件 |
| `PageRequest.of(int page, int size, Direction, String... properties)` | 快速指定單一方向與多個排序欄位 | 所有欄位排序方向相同時適用   |

```java
// 寫法一：不排序，只分頁
Pageable p1 = PageRequest.of(0, 10);

// 寫法二：搭配獨立的 Sort 物件，最靈活
Pageable p2 = PageRequest.of(0, 10, Sort.by("price").descending());

// 寫法三：快速指定方向與欄位
Pageable p3 = PageRequest.of(0, 10, Sort.Direction.ASC, "name", "category");
```

**📐 Sort 進階用法：多欄排序與升降冪混用**

實務上我們經常需要「先照 A 欄位排、A 相同再照 B 欄位排」的複合排序邏輯。`Sort` 物件支援透過 `.and()` 串接多個排序條件，每個條件可分別指定升冪或降冪：

```java
// 先依照 category 升冪排列，若 category 相同，再依照 price 降冪排列
Sort complexSort = Sort.by("category").ascending()
                       .and(Sort.by("price").descending());

Pageable pageable = PageRequest.of(0, 10, complexSort);
```

JPA 會將上述組合轉譯為 SQL 的 `ORDER BY category ASC, price DESC`，完整保留你指定的排序優先順序。

> 💡 **知識補充**
> `Sort.by()` 方法內傳入的字串，例如 `"price"` 或 `"category"`，必須與你的 JPA Entity 類別中的**屬性名稱**完全一致，而**不是資料表欄位名稱**。如果你的 Java 屬性叫做 `productName` 但資料表欄位叫做 `product_name`，這裡應該寫 `Sort.by("productName")`。JPA 會透過 `@Column` 映射自動轉換成正確的 SQL 欄位名稱。

**🚨 常見地雷**

> ⚠️ **常見陷阱 1：頁碼零基索引**
> Spring Data JPA 的底層分頁陣列是從 `0` 開始起算的。而一般使用者認知的首頁是第 `1` 頁。因此撰寫 Controller 接收邏輯時，務必在內部執行扣減換算，以免產生頁碼錯位的問題。

> ⚠️ **常見陷阱 2：未防範非法頁碼**
> 使用者可能手動篡改 URL 中的 `pageNumber` 參數，傳入負數或超過總頁數的數值。如果不在 Controller 中加入邊界防護，JPA 會拋出 `IllegalArgumentException`。建議在換算後加入簡單的防護邏輯：
>
> ```java
> int jpaPage = Math.max(0, pageNumber - 1); // 確保頁碼不會低於 0
> ```

> ⚠️ **常見陷阱 3：Sort 欄位名稱打錯字**
> `Sort.by("pirce")` 中如果把 `price` 拼寫錯誤，JPA 不會在編譯時報錯，而是在**執行時期**才會拋出 `PropertyReferenceException`。由於字串不受編譯器的拼字檢查保護，請務必仔細比對 Entity 中的屬性名稱。

### <a id="CH4-1-2"></a>[2. 前端 Thymeleaf 的進階分頁按鈕邏輯渲染](#toc)

**📍 單元目標**  
於 Thymeleaf 樣板中接收後端傳來的 `Page` 物件，並透過條件判斷與迴圈渲染出完整的分頁導覽列 Pagination 控制元件，包含上一頁、下一頁與數字頁碼按鈕。

**🤔 為什麼需要它**

在上一個單元中，我們已經學會在 Controller 裡建立 `Pageable` 並取得 `Page` 物件，最後透過 `model.addAttribute("page", pageResult)` 把結果丟給前端。但資料送到 Thymeleaf 之後呢？如果只是用 `th:each` 把當頁資料印出來，使用者就只能看到「第一頁」的內容，卻完全沒有辦法切換到其他頁面，等同分頁功能只做了一半。

我們需要在頁面底部渲染出一排**分頁導覽按鈕**，讓使用者能點擊「上一頁」「下一頁」或直接跳到指定頁碼。這排按鈕不能寫死，因為總頁數會隨資料量變動。它必須根據 `Page` 物件提供的資訊**動態生成**：總共幾頁、目前在第幾頁、還有沒有上一頁或下一頁。

**📖 核心概念**

要完成分頁導覽列的渲染，我們需要搭配使用以下幾個 Thymeleaf 語法：

| Thymeleaf 語法                | 用途說明                                           | 對應的分頁場景                                  |
| :---------------------------- | :------------------------------------------------- | :---------------------------------------------- |
| `th:each`                     | 迴圈遍歷集合或數列，為每個元素產生對應的 HTML      | 逐一產生每個頁碼按鈕                            |
| `th:if`                       | 條件為 `true` 時才渲染該元素                       | 有上一頁時才顯示「上一頁」按鈕                  |
| `th:unless`                   | 條件為 `false` 時才渲染該元素，與 `th:if` 互為反義 | 非當前頁碼時才顯示可點擊的連結                  |
| `th:href` 搭配 `@{...()}`     | 產生帶有 Query String 參數的超連結                 | 產生 `/products/page?pageNumber=3` 這類分頁連結 |
| `#numbers.sequence(from, to)` | Thymeleaf 內建的工具方法，產生一段連續整數數列     | 產生 1 到 `totalPages` 的頁碼數列供迴圈使用     |
| `[[${...}]]`                  | 內聯表達式，直接在 HTML 文字中輸出變數值           | 在按鈕文字中顯示頁碼數字                        |

整體的渲染邏輯可以拆解為三個區塊：

1. **上一頁按鈕**：透過 `page.hasPrevious()` 判斷，若目前已是第一頁則不顯示
2. **數字頁碼迴圈**：透過 `#numbers.sequence(1, page.totalPages)` 產生頁碼數列，再逐一判斷是否為當前頁來決定顯示樣式
3. **下一頁按鈕**：透過 `page.hasNext()` 判斷，若目前已是最後一頁則不顯示

**💻 實作範例**

```html
<!-- 資料顯示區塊：迴圈印出當頁的所有商品 -->
<div th:each="prod : ${page.content}">
  <p>品名：[[${prod.name}]] / 價格：[[${prod.price}]]</p>
</div>

<!-- 分頁按鈕導覽列區塊 -->
<div>
  <!-- 上一頁按鈕：如果不是第一頁才顯示 -->
  <a
    th:if="${page.hasPrevious()}"
    th:href="@{/products/page(pageNumber=${currentPage - 1})}"
    >上一頁</a
  >

  <!-- 動態生成中間的數字選單迴圈，例如產生 1 到 5 頁的數列供迴圈依序執行 -->
  <span th:each="pageNum : ${#numbers.sequence(1, page.totalPages)}">
    <!-- 如果是目前所選的頁碼，顯示粗體不加連結 -->
    <strong th:if="${pageNum == currentPage}">[[${pageNum}]]</strong>

    <!-- 如果不是目前頁碼，顯示一般可點擊的連結 -->
    <a
      th:unless="${pageNum == currentPage}"
      th:href="@{/products/page(pageNumber=${pageNum})}"
      >[[${pageNum}]]</a
    >
  </span>

  <!-- 下一頁按鈕：如果不是最後一頁才顯示 -->
  <a
    th:if="${page.hasNext()}"
    th:href="@{/products/page(pageNumber=${currentPage + 1})}"
    >下一頁</a
  >
</div>
```

**🔍 逐行拆解**

1. `th:each="prod : ${page.content}"`  
   `page` 就是 Controller 透過 `model.addAttribute("page", pageResult)` 傳進來的 `Page` 物件。呼叫它的 `getContent()` 方法會取得當頁的資料 `List`，Thymeleaf 會對這個集合逐一迭代，每次將單筆資料指派給 `prod` 變數。

2. `[[${prod.name}]]`  
   這是 Thymeleaf 的**內聯表達式**語法，效果等同於 `th:text="${prod.name}"`，但可以直接寫在 HTML 文字內容中，省去額外的標籤包裹。適合在一段混合文字中插入變數值。

3. `th:if="${page.hasPrevious()}"`  
   呼叫 `Page` 物件的 `hasPrevious()` 方法。若目前是第一頁，此方法回傳 `false`，整個 `<a>` 標籤就不會被渲染到最終的 HTML 中，使用者自然看不到「上一頁」按鈕。

4. `th:href="@{/products/page(pageNumber=${currentPage - 1})}"`  
   `@{...}` 是 Thymeleaf 的 URL 表達式，小括號內的 `(pageNumber=${currentPage - 1})` 會被自動轉換為 Query String 參數。例如當 `currentPage` 為 3 時，最終產生的 HTML 超連結為 `/products/page?pageNumber=2`。

5. `${#numbers.sequence(1, page.totalPages)}`  
   `#numbers` 是 Thymeleaf 內建的數字工具物件，`sequence(1, page.totalPages)` 會產生一個從 1 到總頁數的連續整數陣列。例如總共有 5 頁，就會產生 `[1, 2, 3, 4, 5]`，供外層的 `th:each` 逐一迭代。

6. `th:if` 與 `th:unless` 的搭配  
   在頁碼迴圈內部，我們用這對互為反義的條件判斷來區分「當前頁碼」與「其他頁碼」的顯示方式。當前頁碼以 `<strong>` 粗體呈現且不帶連結，讓使用者一眼辨識所在位置；其他頁碼則渲染為可點擊的 `<a>` 超連結。

> ⚠️ **常見陷阱 1：`#numbers.sequence` 收到 0 會報錯**
> 當資料庫查無任何資料時，`page.totalPages` 的值會是 `0`。此時 `#numbers.sequence(1, 0)` 試圖產生「從 1 到 0」的數列，Thymeleaf 會直接拋出異常。建議在整個分頁導覽列外層加上一層 `th:if` 防護：
>
> ```html
> <!-- 只有在總頁數大於 0 時才渲染分頁按鈕 -->
> <div th:if="${page.totalPages > 0}">
>   <!-- 上一頁、頁碼迴圈、下一頁 ... -->
> </div>
> ```

> ⚠️ **常見陷阱 2：前後端頁碼參數名稱不一致**
> Thymeleaf 連結中寫的參數名稱 `pageNumber=${currentPage - 1}` 必須與 Controller 的 `@RequestParam("pageNumber")` 或 `@RequestParam(defaultValue = "1") Integer pageNumber` **完全吻合**。如果前端寫 `page` 但後端接 `pageNumber`，點擊按鈕後參數就會對不上，導致永遠只顯示預設的第一頁。這種 bug 不會報錯，只會「靜悄悄地失效」，排查起來格外費時。

> 💡 **知識補充**
> 本單元示範的是由伺服器端 Thymeleaf 直接渲染分頁按鈕的 SSR Server-Side Rendering 作法。在現代前後端分離的架構中，後端會改用 `@RestController` 將 `Page` 物件直接序列化為 JSON 回傳，前端再透過 JavaScript 的 `fetch` 或 `Axios` 接收資料後，以 DOM 操作動態渲染分頁元件。兩種架構各有適用場景，完整的前後分離實作將於後續的 RESTful API 課程中進行深度講解。

### <a id="CH4-1-3"></a>[3. 動態查詢實戰：運用 Specification 動態拼裝查詢條件](#toc)

**📍 單元目標**  
學習如何處理使用者僅填寫部分條件的複雜表單，利用 Spring Data JPA 提供的 `Specification` 靈活且強大地動態拼裝 SQL 條件，取代雜亂的 `@Query` 語法。

**🤔 為什麼需要它**  
在真實世界的搜尋表單中，使用者通常不會填滿所有欄位。如果我們在 `@Query` 裡面寫死所有 `WHERE` 條件，很快就會面臨可怕的「義大利麵條程式碼」（例如塞滿 `:keyword IS NULL OR p.name LIKE %:keyword%` 等邏輯）。當條件超過三、四個以上時，不僅極難閱讀更難以擴充維護。我們需要一種如積木般，能視情況動態組裝查詢條件的機制。

**📖 核心概念**

`Specification` 是 Spring Data JPA 提供的一種動態組裝查詢條件介面，其底層封裝了強大的 JPA Criteria API。它將資料庫表單抽象化為物件模型，讓你可以用寫 Java 程式碼的方式來操作 `WHERE` 條件。

| 核心組件               | 運作說明                                                                                                              |
| :--------------------- | :-------------------------------------------------------------------------------------------------------------------- |
| **`Root<T>`**          | 代表查詢的實體物件（如 `Product`），可透過它獲取資料表的各個欄位屬性。                                                |
| **`CriteriaQuery<?>`** | 負責封裝整體的查詢條件、排序與分頁資訊。                                                                              |
| **`CriteriaBuilder`**  | 構建查詢條件的工廠，提供像是 `like()`、`equal()`、`greaterThan()` 等各種運算式。                                      |
| **`Predicate`**        | 代表一個獨立的查詢條件單元，也就是一塊「積木」。多個 `Predicate` 可透過 `and()` 或 `or()` 組裝成完整的 `WHERE` 子句。 |

`CriteriaBuilder` 提供了豐富的方法來建立各種查詢條件，以下列出最常用的方法與其對應的 SQL 語法：

| CriteriaBuilder 方法                | 等效 SQL                   | 使用範例                                          |
| :---------------------------------- | :------------------------- | :------------------------------------------------ |
| `equal(path, value)`                | `column = value`           | `cb.equal(root.get("status"), "active")`          |
| `notEqual(path, value)`             | `column <> value`          | `cb.notEqual(root.get("status"), "deleted")`      |
| `like(path, pattern)`               | `column LIKE pattern`      | `cb.like(root.get("name"), "%手機%")`             |
| `greaterThan(path, value)`          | `column > value`           | `cb.greaterThan(root.get("price"), 1000)`         |
| `greaterThanOrEqualTo(path, value)` | `column >= value`          | `cb.greaterThanOrEqualTo(root.get("price"), 500)` |
| `lessThan(path, value)`             | `column < value`           | `cb.lessThan(root.get("stock"), 10)`              |
| `between(path, lower, upper)`       | `column BETWEEN v1 AND v2` | `cb.between(root.get("price"), 100, 999)`         |
| `isNull(path)`                      | `column IS NULL`           | `cb.isNull(root.get("deletedAt"))`                |
| `isNotNull(path)`                   | `column IS NOT NULL`       | `cb.isNotNull(root.get("email"))`                 |

要使用這項功能，目標 Repository 必須額外繼承 `JpaSpecificationExecutor` 介面，以繼承動態查詢的方法。

**💻 實作範例**

首先，擴充我們的 Repository 介面：

```java
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.JpaSpecificationExecutor;

// 💡 繼承 JpaSpecificationExecutor 以解鎖動態查詢功能
public interface ProductRepository extends JpaRepository<Product, Integer>, JpaSpecificationExecutor<Product> {
    // 🎉 這裡不需再寫複雜的 @Query 動態判斷了！
}
```

接著，建立一個專門產生條件「積木」的配方類別（推薦使用 `static` 方法方便統一呼叫）：

```java
import org.springframework.data.jpa.domain.Specification;
import org.springframework.util.StringUtils;
import jakarta.persistence.criteria.Predicate;
import java.util.ArrayList;
import java.util.List;

public class ProductSpecifications {

    // 產生一個可以動態判斷的 Specification 物件
    public static Specification<Product> filterProducts(String keyword, String category) {

        // 💡 Root 代表 Product 實體，CriteriaBuilder 負責建立條件
        return (root, query, criteriaBuilder) -> {
            // 建立一個集合，專門用來存放準備要拼裝的 WHERE 條件
            List<Predicate> predicates = new ArrayList<>();

            // 若前端有傳入 keyword 且不為空字串，才加入這個模糊搜尋條件
            if (StringUtils.hasText(keyword)) {
                // 等同於 SQL 的: name LIKE %keyword%
                predicates.add(criteriaBuilder.like(root.get("name"), "%" + keyword + "%"));
            }

            // 若前端有傳入 category，才加入這個完全符合條件
            if (StringUtils.hasText(category)) {
                // 等同於 SQL 的: category = category
                predicates.add(criteriaBuilder.equal(root.get("category"), category));
            }

            // 將收集到的所有條件積木用 AND 串接起來並回傳給 JPA 執行
            return criteriaBuilder.and(predicates.toArray(new Predicate[0]));
        };
    }
}
```

**🔍 逐行拆解**

1. `return (root, query, criteriaBuilder) -> { ... };`  
   這是一段 Lambda 表達式，實作了 `Specification<Product>` 介面唯一的 `toPredicate` 方法。JPA 執行查詢時會自動呼叫此方法，並注入三個參數：
   - `root`：代表 `Product` 實體的根物件，可透過 `root.get("欄位名")` 取得對應的資料表欄位
   - `query`：代表整個查詢本身，簡單場景下通常不需要直接操作
   - `criteriaBuilder`：查詢條件的工廠，透過它建立各種 `WHERE` 條件運算式

2. `List<Predicate> predicates = new ArrayList<>();`  
   建立一個集合來暫存所有準備拼裝的條件積木。這就是「積木」的核心——先把零件收集好，最後再一次性拼成成品。

3. `criteriaBuilder.like(root.get("name"), "%" + keyword + "%")`  
   `root.get("name")` 從 `Product` 實體取得 `name` 屬性對應的欄位。`criteriaBuilder.like()` 產生模糊搜尋條件，等效於 SQL 的 `WHERE name LIKE '%手機%'`。前後的 `%` 是 SQL 萬用字元，代表任意數量的任意字元。

4. `criteriaBuilder.equal(root.get("category"), category)`  
   產生完全匹配條件，等效於 SQL 的 `WHERE category = 'phone'`。與 `like()` 不同，`equal()` 要求值完全一致。

5. `criteriaBuilder.and(predicates.toArray(new Predicate[0]))`  
   將收集到的所有條件積木以 `AND` 邏輯串接。`toArray(new Predicate[0])` 是 Java 慣用的集合轉陣列寫法。假設收集了兩個條件，最終產生的 SQL 就會是 `WHERE name LIKE '%手機%' AND category = 'phone'`。

最後，在 Controller 組合並呼叫查詢：

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.jpa.domain.Specification;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import java.util.List;

@Controller
public class ProductSearchController {

    @Autowired
    private ProductRepository productRepository;

    @GetMapping("/products/search")
    public String searchProducts(
            @RequestParam(required = false) String keyword,
            @RequestParam(required = false) String category,
            Model model) {

        // 1. 取得動態拼裝完成的條件積木
        Specification<Product> spec = ProductSpecifications.filterProducts(keyword, category);

        // 2. 將積木傳交給 Repository 的 findAll 方法執行
        List<Product> results = productRepository.findAll(spec);

        model.addAttribute("productList", results);
        return "product-list";
    }
}
```

**🔍 逐行拆解**

1. `@RequestParam(required = false) String keyword`  
   `required = false` 代表此參數為選填。若使用者未傳入，`keyword` 的值為 `null`，而非拋出錯誤。這正是動態查詢的基礎——每個篩選欄位都應設為選填。

2. `ProductSpecifications.filterProducts(keyword, category)`  
   呼叫我們預先定義的靜態方法，將使用者的輸入傳入，取得一個動態組裝完成的 `Specification` 物件。此時條件尚未執行，僅是一張「藍圖」。

3. `productRepository.findAll(spec)`  
   將 `Specification` 藍圖交給 Repository 執行。`JpaSpecificationExecutor` 介面提供的 `findAll(Specification)` 方法會將條件轉譯為實際的 SQL 並送交資料庫。

> 💡 **知識補充**
> `Specification` 還支援直接串聯！如果業務邏輯更加複雜，你可以將每個條件切分成獨立的小方法，再利用鏈式呼叫組裝：
>
> ```java
> // 每個條件獨立為一個方法，更具模組化與可讀性
> Specification<Product> finalSpec = Specification
>         .where(ProductSpecifications.nameContains(keyword))
>         .and(ProductSpecifications.categoryEquals(category))
>         .or(ProductSpecifications.isOnSale());
> ```

**📐 進階整合：Specification 結合 Pageable 分頁查詢**

在真實專案中，「動態篩選 + 分頁」幾乎是標配需求。好消息是，`JpaSpecificationExecutor` 介面已經內建了同時接收 `Specification` 與 `Pageable` 的方法，讓我們一行程式碼就能完成整合：

```java
@GetMapping("/products/search")
public String searchProducts(
        @RequestParam(required = false) String keyword,
        @RequestParam(required = false) String category,
        @RequestParam(defaultValue = "1") Integer pageNumber,
        Model model) {

    // 1. 動態條件積木
    Specification<Product> spec = ProductSpecifications.filterProducts(keyword, category);

    // 2. 分頁規則
    Pageable pageable = PageRequest.of(pageNumber - 1, 10, Sort.by("price").descending());

    // 3. 💡 一行整合！同時進行動態篩選與分頁查詢
    Page<Product> pageResult = productRepository.findAll(spec, pageable);

    model.addAttribute("page", pageResult);
    model.addAttribute("currentPage", pageNumber);
    model.addAttribute("keyword", keyword);       // 回傳搜尋條件供前端回顯
    model.addAttribute("category", category);

    return "product-search-view";
}
```

這段程式碼整合了 4-1-1 的分頁機制與本節的動態查詢，JPA 會自動將 `Specification` 的 `WHERE` 條件與 `Pageable` 的 `LIMIT`/`OFFSET` 合併為一條完整的 SQL 語句執行。

**🚨 常見地雷**

> ⚠️ **常見陷阱 1：忘記空值判斷導致系統崩潰**
> 寫動態條件時，新手常忘記針對傳入的變數進行空值判斷。如果沒有用 `if` 防呆，就直接把使用者未輸入的 `null` 丟進 `criteriaBuilder` 中進行查詢，會引發系統報錯崩潰。推薦善用 Spring 底層附帶的 `StringUtils.hasText()` 來一併嚴格處理空值與全空白字元。

> ⚠️ **常見陷阱 2：忘記繼承 `JpaSpecificationExecutor`**
> 如果你的 Repository 只繼承了 `JpaRepository` 而漏掉 `JpaSpecificationExecutor`，在呼叫 `findAll(spec)` 時編譯器會直接報錯，提示找不到匹配的方法。錯誤訊息可能不太直覺，新手容易誤以為是 Specification 寫錯，請優先檢查 Repository 的繼承宣告。

> ⚠️ **常見陷阱 3：`root.get()` 欄位名稱打錯字**
> 與 4-1-1 提到的 `Sort.by()` 同理，`root.get("pirce")` 如果把 `price` 拼錯，編譯時不會報錯，但執行時會拋出 `IllegalArgumentException`。由於字串不受編譯器保護，請仔細比對 Entity 中的屬性名稱。

> ⚠️ **常見陷阱 4：所有條件皆為空時的邊界情況**
> 當使用者完全不輸入任何篩選條件時，`predicates` 集合會是空的。`criteriaBuilder.and(new Predicate[0])` 在大多數 JPA 實作下會被解讀為「無條件」而回傳所有資料，但為了程式碼的語意明確性與跨實作相容性，建議加入防護：
>
> ```java
> if (predicates.isEmpty()) {
>     return criteriaBuilder.conjunction(); // 等效於 WHERE 1=1，明確地回傳所有資料
> }
> return criteriaBuilder.and(predicates.toArray(new Predicate[0]));
> ```

### 🔁 4-1 章節回顧

1. 進行分頁操作時，傳遞 `PageRequest.of(1, 10)` 給 JPA 時，會抓取出哪一個區段的資料？
   JPA 底層是零基索引，因此會抓取實際為第二頁的十筆資料，新手常在此犯換算疏忽。
2. `Page` 物件內不只包含當頁的資料集合，還提供了哪些強大的輔助方法供前端運用？
   它內建了自動計算總頁數、提供總資料筆數，並具備 `hasNext()`、`hasPrevious()` 等布林值判斷方法，大幅降低前端渲染分頁按鈕的困難度。
3. 在 Thymeleaf 中渲染分頁頁碼按鈕時，我們如何動態產生從 1 到總頁數的數字序列，又需要注意什麼邊界陷阱？
   使用內建工具方法 `#numbers.sequence(1, page.totalPages)` 搭配 `th:each` 迴圈產生頁碼序列。但當資料庫查無資料時 `totalPages` 為 0，`sequence(1, 0)` 會導致 Thymeleaf 拋出異常，因此外層必須加上 `th:if="${page.totalPages > 0}"` 防護。
4. 面對複雜且選填的查詢欄位，我們該如何優雅地處理動態查詢，避免 `@Query` 語法過於臃腫難以維護？
   可以令 Repository 繼承 `JpaSpecificationExecutor`，並利用其 `Specification` 介面搭配 `CriteriaBuilder` 與 Java 條件判斷式，將 SQL 過濾條件如積木般動態拼裝。

---

## <a id="CH4-2"></a>[4-2 資料驗證：Spring Validation](#toc)

### <a id="CH4-2-1"></a>[1. 實作 DTO 欄位驗證 (@NotBlank, @Size, @Pattern)](#toc)

**📍 單元目標**  
將資料驗證邏輯收斂至 DTO 類別當中，並透過標註式宣告取代雜亂無章的 `if-else` 防呆檢查。學完本單元後，你將能夠在任何 DTO 類別上用一行註解就定義出驗證規則，並自訂專屬的中文錯誤提示訊息。

**🤔 為什麼需要它**

想像一下，你正在開發一個會員註冊功能。PM 的需求很明確：帳號 4～12 個字元、密碼至少 6 碼且必須包含英文與數字、Email 必須符合格式。身為 Junior 工程師的你，可能第一直覺會在 Controller 裡面這樣寫：

```java
@PostMapping("/register")
public String register(@ModelAttribute UserRegisterDto dto, Model model) {
    // 😱 驗證邏輯全部擠在 Controller 裡面
    if (dto.getUsername() == null || dto.getUsername().trim().isEmpty()) {
        model.addAttribute("error", "帳號不得為空白");
        return "register-form";
    }
    if (dto.getUsername().length() < 4 || dto.getUsername().length() > 12) {
        model.addAttribute("error", "帳號長度不符");
        return "register-form";
    }
    if (dto.getPassword() == null || !dto.getPassword().matches("^(?=.*[A-Za-z])(?=.*\\d).{6,}$")) {
        model.addAttribute("error", "密碼格式不符");
        return "register-form";
    }
    // ... 還有 Email、手機號碼、地址等更多欄位要繼續寫下去
    return "redirect:/success";
}
```

乍看之下還能接受，但隨著表單欄位越來越多，問題會像滾雪球般放大：

- **程式碼膨脹**
  每多一個欄位就要多寫 3～5 行 `if-else`，一張 10 個欄位的表單光驗證就佔據上百行程式碼
- **重用性為零**  
  如果另一支「編輯個人資料」的 API 也需要驗證同一個 DTO，你只能把整段 `if-else` 複製貼上
- **維護成本高**  
  當產品需求改為「帳號長度放寬到 20 個字元」，你得記得把所有散落各處的魔術數字全部找出來修改

Spring Boot Validation 套件提供了一種優雅的解法：**把驗證規則用註解寫在 DTO 的屬性上方**，就像是在每個欄位旁貼了一張使用說明。Controller 只需一個 `@Valid` 註解就能觸發全部檢查，驗證邏輯與商業邏輯徹底分離。

**📖 核心概念**

使用 Spring Boot Validation 之前，首先必須在 Maven 的 `pom.xml` 加入依賴：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

加入依賴後，即可在 DTO 的屬性上方套用驗證註解。每個註解都支援自訂 `message` 參數來指定驗證失敗時退回的錯誤提示訊息。以下是最常用的驗證註解一覽：

| 常用驗證註解                 | 條件描述與適用型別                                                         | 使用範例                                                          |
| :--------------------------- | :------------------------------------------------------------------------- | :---------------------------------------------------------------- |
| **`@NotBlank`**              | 針對字串，確認不僅不為空值，且剔除掉空白字元後長度必須大於零。             | `@NotBlank(message = "帳號不得為空白")`                           |
| **`@NotNull`**               | 針對數字或物件，確認不能為 null 空值。                                     | `@NotNull(message = "年齡為必填欄位")`                            |
| **`@NotEmpty`**              | 針對字串、集合或陣列，確認不為 null 且長度大於零，但**不會**剔除空白字元。 | `@NotEmpty(message = "標籤列表不得為空")`                         |
| **`@Size(min=X, max=Y)`**    | 驗證字串長度或集合的大小是否落於指定區間。                                 | `@Size(min = 4, max = 12, message = "帳號長度需 4~12 字元")`      |
| **`@Pattern(regexp="...")`** | 強大的正則表達式驗證，用於限制如密碼強度、手機或信箱格式。                 | `@Pattern(regexp = "^09\\d{8}$", message = "手機號碼格式不正確")` |
| **`@Min(X)` / `@Max(Y)`**    | 驗證整數這類別數值型別是否落在合法區間內。                                 | `@Min(value = 0, message = "價格不得為負數")`                     |
| **`@Email`**                 | 驗證字串是否為合法的電子信箱格式。                                         | `@Email(message = "請輸入正確的電子信箱格式")`                    |

> 💡 **知識補充**
> 上方表格中的註解都來自 `jakarta.validation.constraints` 套件，這是 Java 官方的 Bean Validation 標準規範。Spring Boot Validation 底層使用的是 Hibernate Validator 實作引擎，它完整實作了這套標準，因此這些註解在任何支援 Bean Validation 的 Java 框架中都能通用。

**💻 實作範例**

```java
import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Pattern;
import jakarta.validation.constraints.Size;
import lombok.Data;

@Data
public class UserRegisterDto {

    @NotBlank(message = "帳號欄位不得為空白")
    @Size(min = 4, max = 12, message = "帳號長度必須介於 4 到 12 個字元之間")
    private String username;

    @NotBlank(message = "密碼不得為空白")
    // 正則表達式範例，密碼至少包含一個字母和一個數字
    @Pattern(regexp = "^(?=.*[A-Za-z])(?=.*\\d)[A-Za-z\\d]{6,}$",
             message = "密碼格式不符，至少需要 6 碼並包含英數字")
    private String password;

    @NotBlank(message = "電子信箱不得為空白")
    @Email(message = "請輸入正確的電子信箱格式")
    private String email;

    @NotNull(message = "年齡為必填欄位")
    @Min(value = 0, message = "年齡不得為負數")
    private Integer age;
}
```

**🔍 逐行拆解**

1. `@NotBlank(message = "帳號欄位不得為空白")`  
   `@NotBlank` 只能用在 `String` 型別的欄位上。它會先檢查值是否為 `null`，再將字串前後的空白字元全部剃除，最後確認剩餘的長度是否大於零。也就是說，使用者即使只輸入一堆空白鍵，也會被判定為不合法。`message` 參數指定了驗證失敗時的中文提示訊息，如果不寫 `message` 則會使用底層預設的英文訊息。

2. `@Size(min = 4, max = 12, message = "帳號長度必須介於 4 到 12 個字元之間")`  
   `@Size` 檢查字串的「字元數量」是否落在指定的閉區間 `[min, max]` 之內。注意它只算長度，**不會**剔除前後空白。因此我們通常會讓 `@NotBlank` 與 `@Size` 搭檔使用，先由 `@NotBlank` 排除空白干擾，再由 `@Size` 驗證長度。

3. `@Pattern(regexp = "^(?=.*[A-Za-z])(?=.*\\d)[A-Za-z\\d]{6,}$", ...)`  
   `@Pattern` 使用正則表達式進行格式驗證。此範例中 `(?=.*[A-Za-z])` 是一個正向前瞻，確保字串中至少包含一個英文字母；`(?=.*\\d)` 確保至少包含一個數字；`[A-Za-z\\d]{6,}` 限制總長度至少 6 碼。正則表達式雖然強大，但可讀性偏低，建議搭配明確的 `message` 來告訴使用者具體的格式要求。

4. `@Email(message = "請輸入正確的電子信箱格式")`  
   `@Email` 是一個語意化的便捷註解，底層實際上也是透過正則表達式檢查 Email 的基本格式。相較於自己用 `@Pattern` 撰寫 Email 的正則表達式，`@Email` 更加簡潔且不容易出錯。

5. `@NotNull` 搭配 `@Min`  
   `age` 欄位的型別是 `Integer` 包裝類別而非 `int` 基本型別，因此有可能接收到 `null` 值。`@NotNull` 先確保值存在，`@Min(value = 0)` 再確保數值合法。值得注意的是，`@NotBlank` **不能用在非字串型別**上，數值型別的必填檢查必須使用 `@NotNull`。

**🚨 常見地雷**

> ⚠️ **常見陷阱 1：混淆 `@NotBlank`、`@NotNull` 與 `@NotEmpty`**
> 這三個註解名稱極為相似，但適用對象與檢查嚴格度截然不同：
>
> | 註解        | 適用型別         | null | 空字串 `""` | 純空白 `"   "` |
> | :---------- | :--------------- | :--- | :---------- | :------------- |
> | `@NotNull`  | 所有型別         | ❌   | ✅ 通過     | ✅ 通過        |
> | `@NotEmpty` | 字串、集合、陣列 | ❌   | ❌          | ✅ 通過        |
> | `@NotBlank` | 僅 `String`      | ❌   | ❌          | ❌             |

> ⚠️ **常見陷阱 2：`@Pattern` 的跳脫字元加倍**
> 在 Java 字串中，反斜線 `\` 本身就是跳脫字元，因此在正則表達式裡想表達「匹配數字 `\d`」時，必須在 Java 中寫成 `\\d`。常見的新手錯誤是只寫了一個 `\d`，導致編譯器將其視為非法的跳脫字元而報錯。

> ⚠️ **常見陷阱 3：`@Size` 與 `@Min` / `@Max` 的適用對象搞錯**
> `@Size` 用來檢查字串的**字元數量**或集合的**元素數量**，而 `@Min` / `@Max` 用來檢查數值的**大小**。如果你在 `Integer` 欄位上誤用 `@Size`，或在 `String` 欄位上誤用 `@Min`，程式在執行時期會拋出 `UnexpectedTypeException`，提示驗證器找不到匹配的型別處理器。

> ⚠️ **常見陷阱 4：忘記加入 Maven 依賴**
> Spring Boot **不會**自動引入 Validation 依賴。如果 `pom.xml` 中漏掉了 `spring-boot-starter-validation`，所有驗證註解都會被 Spring 靜悄悄地忽略，表單無論填什麼都能通過，這種 bug 極難排查。如果你發現 `@Valid` 完全沒有作用，第一件事就是回頭檢查依賴是否正確引入。

### <a id="CH4-2-2"></a>[2. Controller 攔截 @Valid 與 BindingResult 錯誤解析](#toc)

**📍 單元目標**  
於 Controller 驅動並攔截前述設定的 DTO 驗證機制，並優雅地提取錯誤原因回傳給使用者。學完本單元後，你將能夠在 Controller 中用最精簡的流程觸發驗證、攔截錯誤、決定畫面流向。

**🤔 為什麼需要它**

在上一個單元中，我們已經在 DTO 的每個屬性上貼好了驗證規則的「標籤」。但光是貼標籤並不夠——就像是把行李箱的安檢標準寫在告示牌上，如果沒有安檢員站在入口攔截，乘客仍然能帶著違禁品長驅直入。

`@Valid` 就是那位安檢員。它告訴 Spring 框架：「在呼叫這個方法之前，請先對傳入的 DTO 執行全面檢查。」如果檢查出問題，Spring 會將所有的錯誤資訊打包放入 `BindingResult` 物件中，我們就能根據這個物件決定：是把使用者退回表單重新填寫，還是放行繼續執行商業邏輯。

**📖 核心概念**

在 Controller 方法的 DTO 參數前掛上 `@Valid` 註解即可啟動檢查。緊接著該 DTO 參數的後方，必須**緊隨宣告一個 `BindingResult` 介面參數**，它負責將驗證失敗的詳細資訊保存下來。

`BindingResult` 物件提供了多種實用的方法，讓我們能精準地掌握驗證結果：

| 常用方法                      | 回傳型別            | 說明                                                                             |
| :---------------------------- | :------------------ | :------------------------------------------------------------------------------- |
| `hasErrors()`                 | `boolean`           | 判斷是否存在任何驗證錯誤，通常作為 `if` 判斷的第一道關卡。                       |
| `getErrorCount()`             | `int`               | 取得錯誤總數量，可用於日誌紀錄或除錯追蹤。                                       |
| `getFieldErrors()`            | `List<FieldError>`  | 取得所有**欄位層級**的錯誤清單，每個 `FieldError` 包含欄位名稱與對應的錯誤訊息。 |
| `getFieldError("fieldName")`  | `FieldError`        | 取得指定欄位的第一筆錯誤，若無錯誤則回傳 `null`。                                |
| `getAllErrors()`              | `List<ObjectError>` | 取得所有錯誤，包含欄位層級與物件層級的錯誤。                                     |
| `getFieldErrors("fieldName")` | `List<FieldError>`  | 取得指定欄位的所有錯誤集合，一個欄位可能同時違反多條規則。                       |

> 💡 **知識補充**
> `BindingResult` 繼承自 `Errors` 介面，而 `Errors` 又是 Spring 框架的核心驗證機制之一。在進階應用中，你也可以用它來手動新增自訂的錯誤訊息，例如 `bindingResult.rejectValue("username", "duplicate", "此帳號已被註冊")`，這在像是「帳號重複檢查」這類需要查詢資料庫才能判斷的情境下非常實用。

**💻 實作範例**

```java
import jakarta.validation.Valid;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.validation.BindingResult;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.ModelAttribute;
import org.springframework.web.bind.annotation.PostMapping;

@Controller
public class RegisterController {

    // 💡 處理 GET 請求：第一次進入註冊頁面時，必須準備一個空的 DTO 給 Thymeleaf 綁定
    @GetMapping("/register")
    public String showRegisterForm(Model model) {
        model.addAttribute("userDto", new UserRegisterDto());
        return "register-form";
    }

    // 💡 處理 POST 請求：表單送出後的驗證攔截
    @PostMapping("/register")
    public String processRegister(
            // 1. 加上 @Valid 告訴 Spring 開始驗證 DTO
            @Valid @ModelAttribute("userDto") UserRegisterDto dto,
            // 2. BindingResult 必定要緊跟在 DTO 正後方宣告
            BindingResult bindingResult) {

        // 3. 檢查是否存在任何驗證錯誤
        if (bindingResult.hasErrors()) {
            // 如果驗證失敗，直接中斷商業邏輯，重新轉回註冊表單視圖。
            // 此時 BindingResult 中的錯誤資訊會自動留在 Model 中，
            // Thymeleaf 的 th:errors 可以直接讀取並顯示
            return "register-form";
        }

        // 4. 若無錯誤，呼叫 Service 完成後續註冊邏輯
        // userService.register(dto);
        return "redirect:/success";
    }
}
```

**🔍 逐行拆解**

1. `@GetMapping("/register")` 搭配 `model.addAttribute("userDto", new UserRegisterDto())`  
   使用者第一次造訪註冊頁面時，我們必須預先在 Model 中放入一個**空的** DTO 物件。這是因為 Thymeleaf 的 `th:object="${userDto}"` 在渲染表單時需要一個已存在的物件來綁定，如果找不到這個物件就會直接拋出異常。

2. `@Valid @ModelAttribute("userDto") UserRegisterDto dto`  
   `@ModelAttribute("userDto")` 將表單提交的資料自動綁定到 `UserRegisterDto` 物件上，並以 `"userDto"` 為名稱存入 Model 中。`@Valid` 則在綁定完成後立即觸發 DTO 上所有驗證註解的檢查。執行順序是：先綁定資料 → 再執行驗證。

3. `BindingResult bindingResult`  
   **位置至關重要**。`BindingResult` 必須宣告在被 `@Valid` 標記的 DTO 參數**正後方**。Spring 框架以參數順序來判斷哪個 `BindingResult` 對應哪個驗證目標。如果中間插入了其他參數，Spring 會找不到對應關係而直接拋出例外。

4. `if (bindingResult.hasErrors()) { return "register-form"; }`  
   `hasErrors()` 是最常用的判斷方法。若回傳 `true`，代表至少有一個欄位違反驗證規則。此時我們直接回傳原始的表單頁面名稱，**不使用 `redirect:`**。這是因為直接回傳視圖名稱會保留 Model 中的所有資料，包括 `BindingResult` 的錯誤資訊以及使用者已輸入的內容，Thymeleaf 才能正確地渲染錯誤訊息與回填表單。

5. `return "redirect:/success";`  
   驗證通過後使用 `redirect:` 前綴進行重導向。這裡使用的是 **PRG Post/Redirect/Get 模式**的標準實踐：POST 提交 → 處理成功 → Redirect 到結果頁面。這樣能防止使用者按下 F5 重新整理時重複提交表單。

**🚨 常見地雷**

> ⚠️ **常見陷阱 1：`BindingResult` 的位置放錯**
> 這是新手最高頻的災難性錯誤。如果 `BindingResult` 沒有緊貼在 `@Valid` 標記的 DTO 後面，Spring 會直接拋出 `MethodArgumentNotValidException`，而不是把錯誤收進 `BindingResult`。
>
> ```java
> // ❌ 錯誤示範：Model 插在中間，導致 BindingResult 與 DTO 斷開
> public String process(@Valid @ModelAttribute UserRegisterDto dto,
>                       Model model,
>                       BindingResult result) { ... }
>
> // ✅ 正確示範：BindingResult 緊跟在 DTO 後面
> public String process(@Valid @ModelAttribute UserRegisterDto dto,
>                       BindingResult result,
>                       Model model) { ... }
> ```

> ⚠️ **常見陷阱 2：漏加 `@Valid` 導致驗證完全失效**
> 如果忘記在 DTO 參數前加上 `@Valid`，Spring 會正常綁定資料但**完全跳過驗證流程**。你會發現無論使用者填什麼亂七八糟的資料都能通過，而 `BindingResult.hasErrors()` 永遠是 `false`。這種 bug 不報錯、不拋異常，只會靜悄悄地放行髒資料，非常難以察覺。

> ⚠️ **常見陷阱 3：驗證失敗後誤用 `redirect:`**
> 當驗證失敗需要退回表單時，**絕對不能**使用 `return "redirect:/register";`。因為 `redirect:` 會觸發一次全新的 GET 請求，Model 中的所有資料（包含 `BindingResult` 的錯誤訊息和使用者已輸入的內容）都會被丟棄。使用者會看到一張空白的表單，完全不知道哪裡填錯了。正確做法是直接回傳視圖名稱 `return "register-form";`。

### <a id="CH4-2-3"></a>[3. Thymeleaf 表單錯誤訊息的動態回顯 (Error Rendering)](#toc)

**📍 單元目標**  
將後端抓到的欄位驗證錯誤與提示文案，精準地顯示在前端 HTML 表單的特定欄位旁邊。學完本單元後，你將能夠在 Thymeleaf 中實作欄位級的錯誤紅字回顯，並保留使用者已輸入的內容不被清空。

**🤔 為什麼需要它**

在上一個單元中，我們已經能在 Controller 裡透過 `BindingResult.hasErrors()` 判斷表單是否通過驗證，並在失敗時把使用者退回原始表單頁面。但如果只是單純地退回頁面，使用者會看到一張空白的表單，完全不知道到底是哪個欄位出了問題。

好的使用者體驗應該是：驗證失敗後，**每個出錯的欄位旁邊都精準地顯示紅色的錯誤提示訊息**，同時使用者已經填過的正確內容應該被保留下來，不需要全部重打。這就是所謂的「錯誤回顯」。Thymeleaf 內建了完善的錯誤回顯機制，讓我們幾乎不需要額外撰寫 Java 邏輯就能實現此功能。

**📖 核心概念**

Thymeleaf 的錯誤回顯機制圍繞著以下幾個核心語法協同運作：

| Thymeleaf 語法                   | 用途說明                                                                             | 使用情境                               |
| :------------------------------- | :----------------------------------------------------------------------------------- | :------------------------------------- |
| `th:object="${...}"`             | 在 `<form>` 標籤上綁定整個 DTO 物件，後續的 `*{...}` 表達式都會基於此物件操作。      | 指定表單對應的後端 DTO 模型            |
| `th:field="*{fieldName}"`        | 自動產生 `id`、`name` 屬性，並在驗證失敗退回時自動回填使用者已輸入的值。             | 綁定表單輸入欄位與 DTO 屬性            |
| `th:errors="*{fieldName}"`       | 當指定欄位存在驗證錯誤時，自動將 DTO 註解中定義的 `message` 文字渲染為此元素的內容。 | 在欄位旁邊顯示錯誤提示文字             |
| `#fields.hasErrors('fieldName')` | Thymeleaf 內建的工具方法，回傳布林值判斷指定欄位是否存在驗證錯誤。                   | 搭配 `th:if` 控制錯誤提示元素是否顯示  |
| `#fields.hasAnyErrors()`         | 判斷表單中是否有**任何**欄位存在錯誤，可用於顯示一則全域性的錯誤概要提示。           | 在表單頂部顯示「請修正以下欄位」的提醒 |

整體的錯誤回顯流程可以拆解為四個步驟：

1. **綁定模型**：`th:object` 將 `<form>` 與後端 DTO 綁定
2. **欄位映射**：`th:field` 將每個 `<input>` 與 DTO 的屬性雙向連結
3. **條件判斷**：`#fields.hasErrors()` 判斷該欄位是否有錯誤
4. **訊息渲染**：`th:errors` 輸出驗證註解中定義的 `message` 內容

**💻 實作範例**

```html
<!-- th:object 綁定整個 UserRegisterDto 表單模型 -->
<form th:action="@{/register}" method="post" th:object="${userDto}">
  <!-- 全域性錯誤提醒：若表單有任何欄位驗證失敗，顯示此區塊 -->
  <div
    th:if="${#fields.hasAnyErrors()}"
    style="color: red; margin-bottom: 10px;"
  >
    ⚠️ 表單資料有誤，請修正以下標示的欄位後重新送出。
  </div>

  <div>
    <label>帳號：</label>
    <!-- th:field 自動產生 id="username" name="username" 並回填已輸入的值 -->
    <input type="text" th:field="*{username}" />
    <!-- 動態錯誤回顯：username 存在錯誤時才出現，並顯示 @NotBlank 或 @Size 定義的 message -->
    <span
      th:if="${#fields.hasErrors('username')}"
      th:errors="*{username}"
      style="color: red;"
    >
    </span>
  </div>

  <div>
    <label>密碼：</label>
    <input type="password" th:field="*{password}" />
    <span
      th:if="${#fields.hasErrors('password')}"
      th:errors="*{password}"
      style="color: red;"
    >
    </span>
  </div>

  <div>
    <label>電子信箱：</label>
    <input type="email" th:field="*{email}" />
    <span
      th:if="${#fields.hasErrors('email')}"
      th:errors="*{email}"
      style="color: red;"
    >
    </span>
  </div>

  <div>
    <label>年齡：</label>
    <input type="number" th:field="*{age}" />
    <span
      th:if="${#fields.hasErrors('age')}"
      th:errors="*{age}"
      style="color: red;"
    >
    </span>
  </div>

  <button type="submit">註冊</button>
</form>
```

**🔍 逐行拆解**

1. `th:object="${userDto}"`  
   將整個 `<form>` 與名為 `userDto` 的 Model 屬性綁定。此處的 `userDto` 必須與 Controller 裡 `@ModelAttribute("userDto")` 或 `model.addAttribute("userDto", ...)` 設定的**名稱完全一致**。綁定之後，表單內部就能使用簡潔的星號表達式 `*{...}` 來引用 DTO 的各個屬性。

2. `th:if="${#fields.hasAnyErrors()}"`  
   `#fields` 是 Thymeleaf 專門為表單驗證提供的內建工具物件。`hasAnyErrors()` 方法會檢查 `BindingResult` 中是否存在任何錯誤。我們利用它在表單最頂端顯示一則全域性的提醒文字，讓使用者一眼就知道需要修正資料。

3. `th:field="*{username}"`  
   `th:field` 是 Thymeleaf 表單處理的核心語法糖，它一次完成了**三件事情**：
   - **產生 `id` 屬性**：自動設定為 `id="username"`
   - **產生 `name` 屬性**：自動設定為 `name="username"`，確保表單提交時參數名稱能正確對應 DTO 的屬性
   - **自動回填 `value`**：當驗證失敗退回表單時，會自動將 DTO 物件中該屬性的現有值回填到 `value` 屬性裡，使用者不需要重新輸入

4. `th:if="${#fields.hasErrors('username')}"`  
   針對特定欄位 `username` 檢查是否存在驗證錯誤。只有當該欄位確實有錯時，包含此屬性的 `<span>` 元素才會被渲染到最終的 HTML 中。若無錯誤，這整個 `<span>` 就不會出現在頁面上。

5. `th:errors="*{username}"`  
   當 `username` 欄位的驗證確實失敗時，此屬性會將 DTO 中 `@NotBlank(message = "帳號欄位不得為空白")` 或 `@Size(message = "帳號長度必須介於 4 到 12 個字元之間")` 所定義的 `message` **自動渲染為這個元素的文字內容**。如果同一個欄位同時違反了多條驗證規則，所有的錯誤訊息會被用空白字元分隔後一併顯示。

> 💡 **知識補充**
> `th:field` 的自動回填機制有個例外：對於 `type="password"` 的輸入欄位，基於安全考量，瀏覽器與多數框架都**不會回填密碼值**。因此即使驗證失敗退回表單，密碼欄位仍然會是空白的，這是正常且安全的行為。

**📐 進階整合：搭配 CSS 樣式凸顯錯誤欄位**

在生產環境中，我們通常不僅要顯示錯誤文字，還要讓出錯的輸入欄位本身也有視覺提示，例如紅色邊框：

```html
<div>
  <label>帳號：</label>
  <!-- 💡 透過 th:classappend 動態追加 CSS class，不影響原有的 class -->
  <input
    type="text"
    th:field="*{username}"
    th:classappend="${#fields.hasErrors('username')} ? 'input-error'"
  />
  <span
    th:if="${#fields.hasErrors('username')}"
    th:errors="*{username}"
    class="error-message"
  >
  </span>
</div>
```

搭配的 CSS 樣式：

```css
/* 出錯的輸入欄位加上紅色邊框 */
.input-error {
  border: 2px solid #e74c3c;
  background-color: #fdf2f2;
}

/* 錯誤提示訊息的文字樣式 */
.error-message {
  color: #e74c3c;
  font-size: 0.85em;
  display: block;
  margin-top: 4px;
}
```

`th:classappend` 是一個非常實用的屬性，它會在元素原有的 `class` 基礎上**追加**新的 CSS 類別，而不會覆蓋掉原本的樣式。搭配三元運算子 `條件 ? '類別名稱'`，就能實現「有錯才加紅框」的動態效果。

**🚨 常見地雷**

> ⚠️ **常見陷阱 1：`th:object` 找不到對應的 Model 屬性**
> 如果 Controller 的 `@GetMapping` 方法忘記在 Model 中預先放入空的 DTO 物件，使用者第一次造訪表單頁面時，Thymeleaf 會因為找不到 `${userDto}` 而直接拋出 `BindException` 或 `IllegalStateException`。請務必確認 GET 請求的 Handler 中有執行 `model.addAttribute("userDto", new UserRegisterDto())`。

> ⚠️ **常見陷阱 2：`th:field` 與手動寫 `name` 屬性衝突**
> `th:field` 會自動產生 `id` 和 `name` 屬性。如果你同時手動寫了 `name="username"`，兩者會互相衝突，最終渲染出的 HTML 可能出現重複的 `name` 屬性，導致瀏覽器行為不可預期。使用了 `th:field` 就不要再手動指定 `name`。
>
> ```html
> <!-- ❌ 錯誤：th:field 與手動 name 衝突 -->
> <input type="text" name="username" th:field="*{username}" />
>
> <!-- ✅ 正確：讓 th:field 自動處理 -->
> <input type="text" th:field="*{username}" />
> ```

> ⚠️ **常見陷阱 3：`th:errors` 與 `th:text` 混用**
> `th:errors="*{username}"` 已經會自動將錯誤訊息設定為元素的文字內容。如果你同時又寫了 `th:text="其他文字"`，兩者會互相覆蓋，結果不可預期。`th:errors` 本身就包含了文字渲染的功能，不需要再搭配 `th:text`。

### 🔁 4-2 章節回顧

1. 若要限制使用者輸入的字串不但不能為空，即使隨便打幾個空白鍵也不能通過，應採用何種驗證註解？
   應該使用 `@NotBlank` 註解。相較於 `@NotNull` 只檢查物件存在、`@NotEmpty` 不會剔除空白字元，`@NotBlank` 會嚴格剃除頭尾空白字元後驗證長度，是字串欄位最嚴格的必填驗證。
2. 在使用 Spring Validation 之前，開發者最常犯的遺漏是什麼？
   忘記在 `pom.xml` 中加入 `spring-boot-starter-validation` 依賴。缺少此依賴時，所有驗證註解會被 Spring 靜悄悄地忽略，表單無論填什麼都能通過，且不會拋出任何錯誤訊息。
3. 啟動資料驗證流程時，Controller 方法參數需要注意什麼順序規則？
   宣告負責承接驗證錯誤的 `BindingResult` 參數時，其位置必須**緊接著**在帶有 `@Valid` 註解的目標 DTO 參數之後，中間不得插入任何其他參數，否則框架會因找不到對應關係而直接拋出例外。
4. 驗證失敗後的 Controller 為何不能使用 `redirect:` 回到表單頁面？
   因為 `redirect:` 會觸發全新的 GET 請求，Model 中的所有資料會被丟棄，包含 `BindingResult` 的錯誤資訊與使用者已輸入的內容。必須直接回傳視圖名稱才能保留這些資料供 Thymeleaf 渲染錯誤回顯。
5. Thymeleaf 的 `th:field` 語法糖一次幫開發者完成了哪三件事情？
   自動產生 `id` 屬性與 `name` 屬性以確保表單參數名稱正確對應 DTO，並在驗證失敗退回表單時自動將 DTO 中該屬性的現有值回填到 `value` 屬性裡，實現使用者已輸入內容的保留。

---

## <a id="CH4-3"></a>[4-3 系統日誌與全域例外處理](#toc)

### <a id="CH4-3-1"></a>[1. 告別 System.out：使用 @Slf4j 與 Logback 紀錄層級日誌](#toc)

**📍 單元目標**  
學會以專業的日誌框架取代 `System.out.println`，讓系統運作過程中的每一則訊息都帶有時間戳記、嚴重等級與來源類別資訊，並支援輸出至檔案以便日後追查問題。

**🤔 為什麼需要它**

在學習階段，我們習慣在程式碼裡插入 `System.out.println` 來觀察變數值或確認程式走到哪一行。然而，當專案上線到正式伺服器後，這個做法會面臨嚴重的限制：

```java
// 😱 開發階段的除錯寫法
System.out.println("訂單編號：" + orderId);
System.out.println("商品數量：" + quantity);
System.out.println("發生錯誤！");
```

以上寫法在正式環境中有四個致命問題：

- **沒有時間戳記**  
  每則輸出只有文字內容，你無法得知這則訊息是今天早上 9 點印出的，還是昨天凌晨 3 點印出的
- **沒有嚴重等級**  
  「訂單編號」是一般資訊，「發生錯誤！」是需要緊急處理的例外，但在 `System.out` 的眼中它們毫無分別，全部混在一起
- **無法保存記錄**  
  控制台的輸出一旦超過緩衝區就會被覆蓋，伺服器重啟後更是全部消失，出了問題想回頭追查完全沒有線索
- **無法關閉特定輸出**  
  正式上線後你想隱藏除錯用的 `println`，唯一的辦法就是把程式碼一行一行刪掉或註解掉，改天再需要時又要一行一行加回來

專業的日誌框架能一次解決上述所有問題。它讓你**在程式碼中保留所有的日誌呼叫**，只需透過設定檔調整輸出等級，就能決定哪些訊息該顯示、哪些該隱藏。

**📖 核心概念**

Spring Boot 預設內建 Logback 日誌框架，不需要額外加入任何依賴即可直接使用。搭配 Lombok 提供的 `@Slf4j` 註解，只需在類別上方標注一行，就能自動獲得一個名為 `log` 的日誌紀錄器物件。

日誌系統的核心在於**等級機制**。每則日誌都有一個嚴重等級，框架會根據你設定的「最低輸出等級」來過濾：只有等級**大於或等於**設定值的日誌才會被輸出，較低等級的訊息則會被自動忽略。

以下是五個日誌等級，由低到高排列：

| 等級      | 嚴重程度 | 使用時機                                             | 控制台輸出範例                                                                 |
| :-------- | :------- | :--------------------------------------------------- | :----------------------------------------------------------------------------- |
| **TRACE** | ⬜ 最低  | 極度詳細的追蹤資訊，通常只在深入排查特定問題時開啟。 | `2026-04-21 10:30:00 TRACE c.e.OrderService : 進入 processOrder 方法`          |
| **DEBUG** | 🟦 低    | 開發階段的除錯資訊，如變數值、條件判斷結果。         | `2026-04-21 10:30:00 DEBUG c.e.OrderService : 查詢到商品數量=5`                |
| **INFO**  | 🟩 中    | 系統正常運作的重要節點，如服務啟動、訂單完成。       | `2026-04-21 10:30:00  INFO c.e.OrderService : 訂單 #1024 處理完成`             |
| **WARN**  | 🟨 高    | 潛在的風險警告，系統仍能運作但需要關注。             | `2026-04-21 10:30:00  WARN c.e.OrderService : 庫存不足，商品 #88 僅剩 2 件`    |
| **ERROR** | 🟥 最高  | 發生例外錯誤或功能失敗，需要工程師介入處理。         | `2026-04-21 10:30:00 ERROR c.e.OrderService : 資料庫連線逾時，訂單建立失敗`    |

Spring Boot 預設的輸出等級為 **INFO**，這代表 TRACE 和 DEBUG 等級的訊息在預設情況下都是隱藏的。你可以透過 `application.properties` 調整特定套件的輸出等級：

```properties
# 將 com.example 套件下所有類別的日誌等級設為 DEBUG，開發時可看到更多資訊
logging.level.com.example=DEBUG

# 將 Spring 框架本身的日誌等級提升為 WARN，減少框架內部的雜訊
logging.level.org.springframework=WARN

# 設定日誌輸出至檔案，方便正式環境中回頭追查
logging.file.name=logs/application.log
```

**💻 實作範例**

```java
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

@Slf4j // Lombok 自動產生名為 log 的日誌物件，等同於手動宣告 Logger
@Service
public class OrderService {

    public void processOrder(Integer orderId) {
        // INFO 等級：記錄商業流程的重要節點
        log.info("開始處理訂單，編號：{}", orderId);

        try {
            if (orderId < 0) {
                // WARN 等級：不正常但可能可以容忍的狀況
                log.warn("收到負數訂單編號：{}，流程即將中斷", orderId);
                throw new IllegalArgumentException("無效的訂單編號");
            }

            // DEBUG 等級：開發除錯用的詳細資訊
            log.debug("訂單 {} 通過基本檢查，準備進入付款流程", orderId);

            // ... 模擬後續商業邏輯
            log.info("訂單 {} 處理完成", orderId);

        } catch (Exception e) {
            // ERROR 等級：將例外物件作為最後一個參數傳入，框架會自動印出完整堆疊
            log.error("處理訂單 {} 時發生錯誤", orderId, e);
        }
    }
}
```

**🔍 逐行拆解**

1. `@Slf4j`  
   這是 Lombok 提供的註解，編譯時會自動在類別中注入以下這一行程式碼：
   ```java
   private static final Logger log = LoggerFactory.getLogger(OrderService.class);
   ```
   有了這個註解，我們就不需要手動宣告 Logger 物件，直接使用 `log` 變數即可呼叫各等級的日誌方法。

2. `log.info("開始處理訂單，編號：{}", orderId)`  
   `{}` 是日誌框架專用的**佔位符語法**。框架會在實際輸出時才將 `orderId` 的值填入 `{}` 的位置。這種寫法比字串串接 `"訂單編號：" + orderId` 在效能上更優，因為當該等級的日誌被過濾掉時，字串串接根本不會被執行，避免了不必要的記憶體消耗。

3. `log.warn("收到負數訂單編號：{}，流程即將中斷", orderId)`  
   WARN 等級代表「系統仍能正常運作，但這個狀況值得關注」。在此例中，負數的訂單編號雖然不會讓系統崩潰，但顯然不是正常的請求，可能是前端 bug 或惡意輸入。

4. `log.debug("訂單 {} 通過基本檢查，準備進入付款流程", orderId)`  
   DEBUG 等級的資訊在正式環境中預設是隱藏的，只有開發者主動將等級調降為 DEBUG 時才會出現。因此你可以放心地在程式碼中保留大量的 debug 日誌，不用擔心正式環境被洗版。

5. `log.error("處理訂單 {} 時發生錯誤", orderId, e)`  
   當最後一個參數是 `Exception` 或 `Throwable` 物件時，Logback 會自動識別並印出完整的 Stack Trace 堆疊軌跡。注意這裡的 `e` 不需要對應額外的 `{}`，框架會自動處理。

> 💡 **知識補充**
> 使用 `{}` 佔位符而非字串串接 `+` 是日誌撰寫的最佳實踐。差異在於：
>
> ```java
> // ❌ 字串串接：無論日誌是否會被輸出，這段串接都會執行
> log.debug("查詢結果：" + expensiveObject.toString());
>
> // ✅ 佔位符：如果 DEBUG 等級被過濾，toString() 根本不會被呼叫
> log.debug("查詢結果：{}", expensiveObject);
> ```
>
> 在高流量的系統中，如果大量使用字串串接，這些不必要的物件建立和記憶體分配會累積成明顯的效能損耗。

**🚨 常見地雷**

> ⚠️ **常見陷阱 1：繼續使用 `System.out.println` 混用**
> 在引入日誌框架後，有些新手會在部分地方用 `log.info()`、部分地方仍用 `System.out.println`。這會導致兩套輸出系統並存，`println` 的訊息不會被寫入日誌檔案，也不受等級過濾控制。請在專案中全面統一使用 `log` 物件。

> ⚠️ **常見陷阱 2：在 ERROR 日誌中忘記傳入例外物件**
> ```java
> // ❌ 只記錄了錯誤訊息，沒有堆疊軌跡，無法追查錯誤源頭
> log.error("訂單處理失敗：" + e.getMessage());
>
> // ✅ 將例外物件作為最後一個參數，框架自動印出完整堆疊
> log.error("訂單處理失敗", e);
> ```
> 如果只記錄 `e.getMessage()`，你只會看到一行籠統的錯誤描述。將整個 `e` 物件傳入後，日誌會完整輸出從哪個類別的第幾行開始拋出例外的堆疊軌跡，大幅縮短除錯時間。

> ⚠️ **常見陷阱 3：`@Slf4j` 未生效，找不到 `log` 變數**
> 如果你使用了 `@Slf4j` 但編譯器報錯「找不到 log 變數」，通常是因為專案的 Lombok 設定不完整。請確認 `pom.xml` 中已加入 Lombok 依賴，並且你的 IDE 已安裝 Lombok 外掛。IntelliJ IDEA 使用者可到 `File → Settings → Plugins` 搜尋 Lombok 安裝。

### <a id="CH4-3-2"></a>[2. 認識 @ControllerAdvice 與 @ExceptionHandler](#toc)

**📍 單元目標**  
建立全域性的例外攔截機制，將散落在各個 Controller 中的 `try-catch` 區塊集中管理，讓 Controller 專注於商業邏輯的處理。

**🤔 為什麼需要它**

在沒有全域例外處理的情況下，如果你想在每個 Controller 方法中應對可能的錯誤，程式碼會變成這樣：

```java
@GetMapping("/products/{id}")
public String getProduct(@PathVariable Integer id, Model model) {
    try {
        Product product = productService.findById(id);
        model.addAttribute("product", product);
        return "product-detail";
    } catch (ProductNotFoundException e) {
        model.addAttribute("errorMsg", "找不到該商品");
        return "error-page";
    } catch (Exception e) {
        model.addAttribute("errorMsg", "系統發生錯誤");
        return "error-page";
    }
}
```

如果你有 20 支 Controller 方法，每支都要寫這樣的 `try-catch`，會產生三個嚴重問題：

- **程式碼冗贅**  
  每支方法都要重複撰寫幾乎相同的錯誤處理區塊，大量的 `catch` 淹沒了真正的商業邏輯
- **維護困難**  
  如果設計師改了錯誤頁面的名稱，你得到每一支 Controller 去逐一修改 `return "error-page"`
- **容易遺漏**  
  只要有一支方法忘了寫 `try-catch`，使用者就會看到 Spring Boot 預設的 Whitelabel Error Page 白標錯誤頁面，顯得非常不專業

Spring 提供了 `@ControllerAdvice` 機制，讓你可以建立一個「全域例外攔截中心」。任何 Controller 拋出的未捕捉例外，都會自動交由這個中心統一處理，每支 Controller 方法只需要專心寫商業邏輯就好。

**📖 核心概念**

| 核心元件                               | 角色說明                                                                                                   |
| :------------------------------------- | :--------------------------------------------------------------------------------------------------------- |
| **`@ControllerAdvice`**                | 標註在類別上，宣告該類別為全域的 Controller 輔助類別，Spring 會自動掃描並啟用它。                           |
| **`@ExceptionHandler(XxxException.class)`** | 標註在方法上，指定該方法負責處理哪一種例外類別。當系統拋出匹配的例外時，Spring 會自動呼叫這個方法。   |
| **方法參數 `Exception ex`**            | Spring 會自動將捕捉到的例外物件注入到方法參數中，讓你能取得錯誤訊息或記錄日誌。                             |
| **方法參數 `Model model`**             | 與一般 Controller 方法相同，可透過 Model 傳遞資料給錯誤頁面的 Thymeleaf 模板。                             |

`@ExceptionHandler` 支援精確匹配特定的例外類別。當系統拋出例外時，Spring 會依照以下優先順序尋找匹配的處理方法：

1. **精確匹配**
   先尋找是否有方法宣告了與拋出例外完全一致的類別
2. **父類別匹配**
   若無精確匹配，再向上尋找處理該例外父類別的方法
3. **最終兜底**：如果宣告了 `@ExceptionHandler(Exception.class)`，它會攔截所有未被前面方法匹配到的例外

> 💡 **知識補充**
> `@ControllerAdvice` 預設會攔截**所有** Controller 的例外。如果你只想針對特定套件下的 Controller 生效，可以透過參數縮小範圍：
>
> ```java
> // 只攔截 com.example.web 套件下的 Controller
> @ControllerAdvice(basePackages = "com.example.web")
> ```

**💻 實作範例**

假設我們的業務場景中可能遇到兩種不同的例外情境：查詢不存在的資源，以及未預期的系統內部錯誤。我們可以針對不同的例外類別分別處理：

```java
import lombok.extern.slf4j.Slf4j;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;

@Slf4j
@ControllerAdvice
public class GlobalExceptionHandler {

    // 1. 處理「找不到資源」的特定例外
    @ExceptionHandler(IllegalArgumentException.class)
    public String handleNotFound(IllegalArgumentException ex, Model model) {
        // 記錄 WARN 等級：這不是系統 bug，而是使用者請求了不存在的資源
        log.warn("收到不合法的請求參數：{}", ex.getMessage());

        model.addAttribute("errorMsg", ex.getMessage());
        return "error-page";
    }

    // 2. 兜底處理所有其他未預期的例外
    @ExceptionHandler(Exception.class)
    public String handleAllExceptions(Exception ex, Model model) {
        // 記錄 ERROR 等級：這是真正的系統異常，需要工程師介入
        log.error("系統發生未預期的例外", ex);

        // 傳給前端的訊息不應包含任何技術細節
        model.addAttribute("errorMsg", "系統暫時無法處理您的請求，請稍後再試");
        return "error-page";
    }
}
```

**🔍 逐行拆解**

1. `@ControllerAdvice`  
   將這個類別註冊為全域的 Controller 輔助元件。Spring 啟動時會自動掃描找到它，並將其納入例外處理的責任鏈中。所有 Controller 拋出的未捕捉例外，都會被導向到這個類別的對應方法中處理。

2. `@ExceptionHandler(IllegalArgumentException.class)`  
   宣告此方法專門負責處理 `IllegalArgumentException` 類型的例外。由於這是精確匹配，當 Controller 拋出 `IllegalArgumentException` 時，Spring 會**優先**呼叫這個方法，而不是下方的兜底方法。

3. `log.warn("收到不合法的請求參數：{}", ex.getMessage())`  
   使用 WARN 而非 ERROR 等級。這是一個設計上的判斷：使用者查詢不存在的資料屬於「可預期的異常」，並非系統 bug，因此以 WARN 等級記錄足矣。將 ERROR 保留給真正的系統故障，能避免日誌中的嚴重錯誤被大量的正常異常淹沒。

4. `@ExceptionHandler(Exception.class)`  
   `Exception` 是所有例外類別的父類別，因此這個方法會攔截所有**未被前面方法精確匹配**到的例外。它扮演的角色是「安全網」，確保沒有任何例外能逃過處理而直接顯示 Spring 預設的白標錯誤頁面。

5. `model.addAttribute("errorMsg", "系統暫時無法處理您的請求，請稍後再試")`  
   傳遞給前端的錯誤訊息必須經過「去技術化」處理。使用者不需要知道是 `NullPointerException` 還是 `DataAccessException`，他們只需要被告知「目前有問題，我們正在處理」。具體的技術細節透過 `log.error()` 記錄在伺服器日誌中，供工程師事後排查。

### <a id="CH4-3-3"></a>[3. 實作：錯誤頁面模板與自訂例外類別](#toc)

**📍 單元目標**  
準備面向使用者的友善錯誤頁面模板，並學習如何定義自訂例外類別，讓例外處理的語意更加清晰。

**🤔 為什麼需要它**

上一個單元的範例中，我們使用了 `IllegalArgumentException` 來代表「找不到資源」的情境。但 `IllegalArgumentException` 是 Java 的通用例外類別，它可能在各種不同場景下被拋出，語意不夠明確。如果你的系統同時有「商品不存在」、「使用者不存在」、「訂單不存在」等多種情境，全部都用 `IllegalArgumentException` 就很難在 `@ExceptionHandler` 中做出有意義的區分。

自訂例外類別讓你能為每一種業務錯誤賦予清晰的名稱與語意，就像是為不同的錯誤貼上專屬的標籤，讓例外處理機制能準確地辨識並回應。

**📖 核心概念**

自訂例外類別只需要繼承 Java 的 `RuntimeException`，並透過建構子傳入自訂的錯誤訊息。建議為每種業務場景的錯誤建立獨立的例外類別：

| 自訂例外類別名稱          | 對應的業務場景                     | 建議的日誌等級 |
| :----------------------- | :--------------------------------- | :------------- |
| `ResourceNotFoundException` | 查詢的資料不存在                 | WARN           |
| `DuplicateDataException`   | 新增的資料已存在，違反唯一性限制 | WARN           |
| `BusinessLogicException`   | 業務邏輯檢查不通過               | WARN           |

**💻 實作範例**

首先，定義自訂例外類別：

```java
// 自訂例外類別，代表「查詢的資源不存在」
public class ResourceNotFoundException extends RuntimeException {

    // 透過建構子傳入自訂的錯誤訊息
    public ResourceNotFoundException(String message) {
        super(message);
    }
}
```

在 Service 層中拋出自訂例外：

```java
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Slf4j
@Service
public class ProductService {

    @Autowired
    private ProductRepository productRepository;

    public Product findById(Integer id) {
        // 查詢資料庫，若找不到則拋出自訂例外
        return productRepository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException(
                        "找不到編號為 " + id + " 的商品"
                ));
    }
}
```

接著，在全域例外處理器中新增針對自訂例外的攔截方法：

```java
import lombok.extern.slf4j.Slf4j;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;

@Slf4j
@ControllerAdvice
public class GlobalExceptionHandler {

    // 1. 處理自訂的「資源不存在」例外
    @ExceptionHandler(ResourceNotFoundException.class)
    public String handleResourceNotFound(ResourceNotFoundException ex, Model model) {
        log.warn("資源查詢未果：{}", ex.getMessage());

        model.addAttribute("errorMsg", ex.getMessage());
        return "error-page";
    }

    // 2. 兜底處理所有其他未預期的例外
    @ExceptionHandler(Exception.class)
    public String handleAllExceptions(Exception ex, Model model) {
        log.error("系統發生未預期的例外", ex);

        model.addAttribute("errorMsg", "系統暫時無法處理您的請求，請稍後再試");
        return "error-page";
    }
}
```

最後，準備面向使用者的友善錯誤頁面 `error-page.html`：

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
  <title>錯誤提示</title>
</head>
<body>
  <div>
    <h1>Oops！發生了一些問題</h1>
    <p>很抱歉，系統目前無法完成您的請求。</p>
    <p>錯誤資訊：<span th:text="${errorMsg}"></span></p>
    <a th:href="@{/}">返回首頁</a>
  </div>
</body>
</html>
```

**🔍 逐行拆解**

1. `public class ResourceNotFoundException extends RuntimeException`  
   繼承 `RuntimeException` 而非 `Exception`。這是 Spring 框架的慣例做法，因為 `RuntimeException` 屬於非受檢例外 Unchecked Exception，呼叫端不需要在方法簽章上宣告 `throws`，也不需要強制用 `try-catch` 包裹，程式碼更加簡潔。

2. `.orElseThrow(() -> new ResourceNotFoundException("找不到編號為 " + id + " 的商品"))`  
   `Optional.orElseThrow()` 是 JPA `findById()` 回傳 `Optional` 時的標準處理手法。當 `Optional` 為空時，Lambda 中的程式碼會被執行並拋出指定的例外。這比先判斷 `if (result == null)` 再手動拋出例外要更加簡潔。

3. `@ExceptionHandler(ResourceNotFoundException.class)` 的匹配優先權  
   當 Service 拋出 `ResourceNotFoundException` 時，Spring 會先檢查是否有方法精確匹配此類別。由於我們宣告了專門的處理方法，它會被優先呼叫。如果將這個方法刪除，同樣的例外就會回落到 `@ExceptionHandler(Exception.class)` 的兜底方法中被處理。

4. WARN vs ERROR 的日誌等級選擇  
   「找不到資源」屬於可預期的業務例外，使用者輸入了不存在的 ID 並不是系統 bug，所以用 WARN 等級就夠了。只有真正意料之外的錯誤才值得用 ERROR 等級，這樣工程師在檢查日誌時能快速篩選出需要優先處理的問題。

> 💡 **知識補充**
> 在進階的前後端分離架構中，`@ControllerAdvice` 可以替換為 `@RestControllerAdvice`，搭配 `@ResponseStatus` 註解來回傳帶有正確 HTTP 狀態碼的 JSON 錯誤回應，而不是轉導到 HTML 錯誤頁面。這部分將在後續的 RESTful API 課程中深入介紹。

**🚨 常見地雷**

> ⚠️ **常見陷阱 1：把原始的 Stack Trace 暴露給前端**
> 將例外物件的 `e.toString()` 或 `e.getStackTrace()` 直接傳給前端頁面顯示，是非常嚴重的安全漏洞。攻擊者可以從堆疊軌跡中分析出伺服器使用的框架版本、資料庫類型、檔案路徑等敏感資訊。務必確保傳給 Model 的只有經過處理的友善文字。

> ⚠️ **常見陷阱 2：`@ExceptionHandler` 方法的宣告順序無法控制優先權**
> 有些新手以為把 `@ExceptionHandler(Exception.class)` 寫在類別的最前面就能「先執行」。事實上，Spring 的例外匹配是根據**類別繼承的精確度**來決定優先順序，與方法在原始碼中的順序無關。精確匹配的子類別永遠優先於父類別。

> ⚠️ **常見陷阱 3：自訂例外類別繼承了 `Exception` 而非 `RuntimeException`**
> 如果你的自訂例外繼承的是 `Exception`，它就變成了受檢例外 Checked Exception。這意味著所有呼叫端都必須在方法簽章上明確寫出 `throws ResourceNotFoundException`，否則編譯器會報錯。在 Spring 框架中，慣例做法是繼承 `RuntimeException`，保持程式碼的簡潔性。

### 🔁 4-3 章節回顧

1. 為什麼正式專案中應該全面使用日誌框架，而不是繼續使用 `System.out.println`？
   因為 `System.out.println` 不具備時間戳記、嚴重等級區分、檔案保存等能力，輸出的內容在伺服器重啟後會全部消失，也無法透過設定檔控制哪些訊息要顯示或隱藏，不利於正式環境的問題追查。
2. 日誌的 `{}` 佔位符語法相較於字串串接 `+` 有什麼效能優勢？
   使用 `{}` 佔位符時，如果該等級的日誌被過濾而不輸出，後方的參數根本不會被轉換為字串，避免了不必要的字串建立與記憶體消耗。字串串接則是無論日誌是否輸出都會先執行串接運算。
3. `@ControllerAdvice` 搭配 `@ExceptionHandler` 解決了什麼實務問題？
   解決了例外處理邏輯散落在每個 Controller 方法中的冗贅問題。透過全域例外攔截機制，所有 Controller 的未捕捉例外都會被統一導向至專責的處理類別，讓 Controller 只需專注於商業邏輯。
4. 當同一個 `@ControllerAdvice` 類別中同時有 `@ExceptionHandler(IllegalArgumentException.class)` 和 `@ExceptionHandler(Exception.class)` 時，拋出 `IllegalArgumentException` 會觸發哪一個？
   會觸發精確匹配的 `IllegalArgumentException` 處理方法。Spring 的匹配機制是根據例外類別的繼承精確度決定優先順序，子類別匹配永遠優先於父類別。
5. 為什麼自訂例外類別建議繼承 `RuntimeException` 而非 `Exception`？
   因為繼承 `RuntimeException` 的例外屬於非受檢例外，呼叫端不需要在方法簽章中宣告 `throws`，也不需要強制用 `try-catch` 包裹，程式碼更加簡潔。這是 Spring 框架中的慣例做法。
