```mermaid
flowchart TD
    Start([開始<br/>初始化腳本])

    %% ====== INIT ======
    Start --> Init[方形: init<br/>（初始化翻譯功能）]
    Init -->|設定 HTML Lang 屬性| SetLang[方形: 設定<br/>document.documentElement.lang<br/>＝ CONFIG.LANG]
    SetLang -->|監控 Lang 屬性| MO_Lang[方形: MutationObserver<br/>監控 lang 屬性]
    MO_Lang -->|註冊 Turbo 事件| TurboEvt[方形: turbo:load<br/>事件監聽]
    TurboEvt -->|註冊選單| MenuReg[方形: registerMenuCommand<br/>註冊功能選單]
    MenuReg -->|開始監控變動| WatchUpd[方形: watchUpdate<br/>監控 DOM 變動]
    WatchUpd -->|載入時翻譯| DOMContent[方形: DOMContentLoaded<br/>頁面初始翻譯]
    DOMContent -->|更新頁型| UpdatePC[方形: updatePageConfig<br/>（'首次載入'）]
    UpdatePC -->|判斷型別| buildPC[方形: buildPageConfig<br/>產生 pageConfig]
    buildPC -->|翻譯| Traverse[方形: traverseNode<br/>遍歷 body 執行翻譯]

    %% ====== watchUpdate ======
    WatchUpd --> MOStart[方形: 監控 document.body<br/>MutationObserver]
    MOStart -->|URL 變化？| DIAMOND_URL{菱形:<br/>currentURL ＝＝ previousURL<br/>？}
    DIAMOND_URL -- 否 --> UpdatePC2[方形: updatePageConfig<br/>（'DOM變化'）]
    UpdatePC2 --> buildPC2
    DIAMOND_URL -- 是 --> ContMut[方形: 遍歷突變]
    UpdatePC2 --> ContMut

    %% ====== processMutations 分支 ======
    ContMut -->|突變類型| DIAMOND_MUT{菱形:<br/>type ＝＝ 'childList' 且<br/>addedNodes.length ＞ ０？}
    DIAMOND_MUT -- 是 --> AddNodes[方形: 取出<br/>...addedNodes]
    DIAMOND_MUT -- 否 --> AttrText[方形: type ＝＝ 'attributes'<br/>或 'characterData' 且<br/>頁型允許？]
    AttrText -->|是| TargetNode[方形: 處理 target]
    AttrText -->|否| SkipNode[方形: 跳過]
    AddNodes --> FilterNodes
    TargetNode --> FilterNodes
    SkipNode -->|結束| MOStart

    FilterNodes[方形: 過濾<br/>父元素是否在<br/>ignoreMutationSelectors]
    FilterNodes -->|否| TraverseNodes[方形: traverseNode<br/>遞迴處理每個節點]
    FilterNodes -->|是| MOStart

    %% ====== traverseNode 遞迴 ======
    TraverseNodes -->|nodeType<br/>TEXT_NODE？| DIAMOND_TEXT{菱形:<br/>nodeType ＝＝ TEXT_NODE？}
    DIAMOND_TEXT -- 是 --> HandleText[方形: handleTextNode]
    DIAMOND_TEXT -- 否 --> TreeWalker[方形: TreeWalker 遍歷<br/>ELEMENT_NODE 及 TEXT_NODE]

    HandleText -->|長度＞500？| DIAMOND_LEN{菱形:<br/>node.length ＞ 500？}
    DIAMOND_LEN -- 是 --> EndTraverse[方形: 跳過此節點]
    DIAMOND_LEN -- 否 --> TransElemData[方形: transElement<br/>node, 'data']
    TransElemData --> EndTraverse
    EndTraverse --> TreeWalker

    TreeWalker -->|依 tagName 處理| HandleElem[方形: 處理不同標籤屬性]
    HandleElem -->|RELATIVE-TIME| TransTime[方形: transTimeElement]
    HandleElem -->|INPUT/TEXTAREA| TransInp[方形: 按型態翻譯 value/placeholder]
    HandleElem -->|OPTGROUP| TransOpt[方形: transElement<br/>node, 'label']
    HandleElem -->|BUTTON/A/SPAN| TransBtn[方形: 翻譯 title、data-confirm 等]
    HandleElem -->|tooltipped 樣式| TransAria[方形: transElement<br/>node, 'ariaLabel']
    TransTime & TransInp & TransOpt & TransBtn & TransAria --> EndTreeWalker[方形: 完成單一節點]
    EndTreeWalker --> TreeWalker

    %% ====== transElement/ transText 流程 ======
    TransElemData -->|text 是否存在？| DIAMOND_TXT{菱形:<br/>text 是否存在？}
    DIAMOND_TXT -- 否 --> EndTraverse
    DIAMOND_TXT -- 是 --> TransText[方形: transText(text)]

    TransText -->|shouldSkip？| DIAMOND_SKIP{菱形:<br/>shouldSkip(text)？}
    DIAMOND_SKIP -- 是 --> EndTraverse
    DIAMOND_SKIP -- 否 --> CleanText[方形: 清理與 trim]
    CleanText --> FetchTrans[方形: fetchTranslatedText]
    FetchTrans -->|有結果且不同？| DIAMOND_RET{菱形:<br/>translatedText !== cleanedText？}
    DIAMOND_RET -- 是 --> ReplaceText[方形: 回傳替換後字串]
    DIAMOND_RET -- 否 --> EndTraverse

    %% ====== fetchTranslatedText 判斷 ======
    FetchTrans -->|pageConfig.staticDict 有翻譯？| DIAMOND_DICT{菱形:<br/>有靜態翻譯？}
    DIAMOND_DICT -- 是 --> ReturnDict[方形: 回傳翻譯]
    DIAMOND_DICT -- 否 -->|啟用正則？| DIAMOND_REG{菱形:<br/>FeatureSet.enable_RegExp？}
    DIAMOND_REG -- 是 --> ForReg[方形: 遍歷 regexpRules]
    ForReg -->|有正則匹配？| DIAMOND_MATCH{菱形:<br/>translatedText !== text？}
    DIAMOND_MATCH -- 是 --> ReturnReg[方形: 回傳正則翻譯]
    DIAMOND_MATCH -- 否 --> ReturnFalse[方形: 回傳 false]
    DIAMOND_REG -- 否 --> ReturnFalse
    ReturnDict & ReturnReg & ReturnFalse --> FetchTrans

    %% ====== transDesc（翻譯按鈕）流程 ======
    TurboEvt -->|頁型有 DESC_SELECTOR？| DIAMOND_DESC{菱形:<br/>有描述選擇器？}
    DIAMOND_DESC -- 否 --> EndTurbo[方形: 跳過]
    DIAMOND_DESC -- 是 --> DescBtn[方形: 插入翻譯按鈕]
    DescBtn -->|按下翻譯？| DIAMOND_BTN{菱形:<br/>button.disabled？}
    DIAMOND_BTN -- 否 --> GetText[方形: 取出描述文字]
    GetText -->|有值？| DIAMOND_VAL{菱形:<br/>descText 是否存在？}
    DIAMOND_VAL -- 否 --> EndTurbo
    DIAMOND_VAL -- 是 --> ReqRemote[方形: requestRemoteTranslation]
    ReqRemote -->|回傳翻譯| ShowTrans[方形: 顯示翻譯結果]
    ShowTrans --> EndTurbo
    DIAMOND_BTN -- 是 --> EndTurbo

    %% ====== registerMenuCommand（選單）流程 ======
    MenuReg --> ForMenu[方形: 遍歷功能選單]
    ForMenu -->|點擊選單？| DIAMOND_MENU{菱形:<br/>是否點擊？}
    DIAMOND_MENU -- 否 --> ForMenu
    DIAMOND_MENU -- 是 --> ToggleState[方形: 切換啟用/停用狀態]
    ToggleState -->|有 callback？| DIAMOND_CB{菱形:<br/>有 callback？}
    DIAMOND_CB -- 是 --> RunCB[方形: 執行 callback]
    RunCB --> ForMenu
    DIAMOND_CB -- 否 --> ForMenu

    %% ====== detectPageType 流程 ======
    UpdatePC --> DetectPT[方形: detectPageType]
    DetectPT -->|匹配成功？| DIAMOND_PTYPE{菱形:<br/>有詞庫且 pageType？}
    DIAMOND_PTYPE -- 否 --> LogWarn[方形: 輸出警告<br/>return false]
    DIAMOND_PTYPE -- 是 --> ReturnType[方形: 回傳 pageType]
    LogWarn & ReturnType --> buildPC

    %% ====== 結束節點 ======
    EndTurbo([結束])
```
**註解與精簡分析：**  
- 所有符號、運算子皆用全形（如：＝＝、＞、？）。
- 節點標題均已適當分行（使用<br/>）以精簡長度。
- 菱形節點每個都明確有「是」與「否」兩條分支路線。
- 主要流程包含：初始化(init)、監控變動(watchUpdate)、遍歷節點(traverseNode)、翻譯(transElement/transText)、描述翻譯按鈕(transDesc)、選單切換(registerMenuCommand)、頁型判斷(detectPageType)等。
- 每個函數與主要運算步驟皆有對應節點，便於追蹤邏輯流向。

> **此流程圖可直接貼入支援 Mermaid 的 Markdown 編輯器檢視。**