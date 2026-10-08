# 재정정보 탭 서버 페이징 — 코드 변경 명세 (1-5 ~ 4)

표기: **[변경]** / **[추가]** / **[주석처리]**

---

## 1-5. 탭 목록·건수 API + 헬퍼 [추가]

`NotificationController.java`

```java
/* 재정정보 목록 API : 현재 페이지 분량만 반환 */
@RequestMapping(value = "/notification/selectInfoList.do", method = RequestMethod.GET)
public String selectInfoList(HttpServletRequest request, ModelMap model) throws Exception {
    int size = parseIntOrDefault(request.getParameter("size"), 10, 10, 50);
    int page = parseIntOrDefault(request.getParameter("page"), 1, 1, 100000);

    List<MngNotificationVO> items = selectInfoTabItems(request);
    int from = Math.min((page - 1) * size, items.size());
    int to   = Math.min(from + size, items.size());

    model.addAttribute("result", new ArrayList<MngNotificationVO>(items.subList(from, to)));
    return "jsonView";   // ※ 공통 API와 같은 JSON 반환 방식
}


/* 재정정보 건수 API : 선택 탭 기준 건수 */
@RequestMapping(value = "/notification/selectInfoListCnt.do", method = RequestMethod.GET)
public String selectInfoListCnt(HttpServletRequest request, ModelMap model) throws Exception {
    model.addAttribute("result", selectInfoTabItems(request).size());
    return "jsonView";
}


/* 공통 서비스로 재정정보 전체 조회 → 탭 기준 분류 */
private List<MngNotificationVO> selectInfoTabItems(HttpServletRequest request) throws Exception {
    boolean bizTab = "biz".equals(toInfoTab(request.getParameter("tab")));

    // 기존 info_view.do와 같은 방식으로 MngNotificationVO 구성
    MngNotificationVO body = new MngNotificationVO();
    body.setNtcnYardTySeCd(INFO_SE_CD);   // seCd : 요청값 대신 고정 (기존 코드와 같은 setter)

    // ▼▼ 공통 selectBoardListFile.do에서 검색어·페이징 값을 MngNotificationVO에 넣는 줄을
    //    그대로 복사한 뒤, page는 1 / size는 INFO_FETCH_MAX 로만 바꿀 것 (아래 setter 이름은 예시)
    body.setSelScope(request.getParameter("selScope"));
    body.setSubSech(request.getParameter("subSech"));
    body.setPage("1");
    body.setSize(String.valueOf(INFO_FETCH_MAX));
    // ▲▲

    // 공통 selectBoardListFile.do와 같은 서비스 메서드 호출 (호출 후 후처리가 있으면 동일하게)
    List<MngNotificationVO> all = boardService.selectBoardListFile(body);

    List<MngNotificationVO> tabItems = new ArrayList<MngNotificationVO>();
    for (MngNotificationVO item : all) {
        String title = item.getNtcnYardSjNm();
        boolean isBiz = title != null && title.contains(BIZ_KEYWORD);
        if (isBiz == bizTab) {
            tabItems.add(item);
        }
    }
    return tabItems;
}


/* 헬퍼 */
private static String toInfoTab(String v) {
    return "etc".equals(v) ? "etc" : "biz";
}

private static int parseIntOrDefault(String v, int def, int min, int max) {
    try {
        int n = Integer.parseInt(v);
        return (n < min || n > max) ? def : n;
    } catch (NumberFormatException e) {
        return def;
    }
}
```

---

## 2. 프론트: `info.jsp`

### 2-1. 탭 마크업 [변경]

```jsp
<nav class="link_tab len_5" id="infoTab" style="padding-bottom:0px !important; margin-bottom:0px !important;">
    <ul class="inner">
        <li data-tab="biz" class="${tab eq 'biz' ? 'current' : ''}">
            <a href="#" ${tab eq 'biz' ? 'title="선택됨"' : ''}>업무추진비</a>
        </li>
        <li data-tab="etc" class="${tab eq 'etc' ? 'current' : ''}">
            <a href="#" ${tab eq 'etc' ? 'title="선택됨"' : ''}>그 외 기타정보</a>
        </li>
    </ul>
</nav>
```

### 2-2. `#searchForm` [변경]

```jsp
<form id="searchForm" method="POST">
    <input type="hidden" name="selScope"       id="selScope" value="<c:out value='${selScope}'/>">
    <input type="hidden" name="subSech"        id="subSech"  value="<c:out value='${subSech}'/>">
    <input type="hidden" name="size"           id="size"     value="<c:out value='${size}'/>">
    <input type="hidden" name="page"           id="page"     value="<c:out value='${page}'/>">
    <input type="hidden" name="seCd"           id="seCd"     value="0002">
    <%-- [추가] --%>
    <input type="hidden" name="tab"            id="tab"      value="<c:out value='${tab}'/>">
    <input type="hidden" name="ntcnYardOrdrNo" id="ntcnYardOrdrNo" value="">
</form>
```

### 2-3. 기존 스크립트 [주석처리]

기존 `<script>(function(){ 'use strict'; ... })();</script>` 블록 전체를 `<%-- ... --%>`로 감쌉니다.
대상: `loadAllPages`, `classify`, `renumber`, `render`, `renderPaging`, `goPaging_PagingViewBiz/Etc`, resize 핸들러.

```jsp
<%-- [주석처리] 기존 클라이언트 분류/페이징 스크립트 전체 (롤백 대비, 안정화 후 삭제)
<script>
(function(){
    'use strict';
    var VIEW_URL = '/kor/notification/info_view.do';
    ...
    function loadAllPages(triggerFn, onDone){ ... }
    function classify(items){ ... }
    function renumber(items){ ... }
    function loadBoard() { ... }
    function runSearch() { ... }
    function afterLoad(items){ ... }
    function render(tab) { ... }
    function renderPaging(tab, total, page){ ... }
    window.goPaging_PagingViewBiz = function(p){ ... };
    window.goPaging_PagingViewEtc = function(p){ ... };
    function bindEvents(){ ... $(window).on('resize', ...) ... }
    jQuery(function(){ ... });
})();
</script>
--%>
```

### 2-4. 신규 스크립트 [추가]

```html
<script src="/js/cmn/paging.js"></script>
<script>
// board.js boardOutputList()가 0건일 때 전역 boardList.empty()를 호출하므로 반드시 전역
var boardList;
var viewLink;
// board.js setPaging()이 'PagingView' 토큰으로 페이징 링크를 생성
var goPaging_PagingView = function (cPage) { boardList.setPage(cPage); };

jQuery(function () {
    var $list = $('#list');

    boardList = new getBoardData({
        url:      '/kor/notification/selectInfoList.do',      // 신규 API (board.js 수정 없음)
        urlCount: '/kor/notification/selectInfoListCnt.do',
        viewer:   '/kor/notification/info_view.do',
        complete: function (data) {
            $list.html(boardOutputList('textBoard', data.result));
        }
    });

    viewLink = function (id) { boardList.viewLink(id); };
    boardList.getList();

    /* 탭 전환 */
    $('#infoTab').on('click', 'li[data-tab] > a', function (e) {
        e.preventDefault();
        var $a  = $(this);
        var $li = $a.parent();
        var tab = $li.data('tab');
        if (tab === $('#tab').val()) return;

        $li.addClass('current').siblings().removeClass('current');
        $li.siblings().children('a').removeAttr('title');
        $a.attr('title', '선택됨');

        $('#tab').val(tab);
        $('#page').val(1);
        // [정책 선택] 탭 전환 시 검색 초기화가 필요하면 아래 주석 해제
        // $('#subSech').val(''); $('#searchString').val('');

        // type:'search' → board.js가 캐시한 건수를 무시하고 새 탭 기준으로 재조회
        boardList.getList({ type: 'search' });
    });

    /* 검색 : 현재 탭 안에서 검색 */
    $('#boardSearchForm').on('submit', function (e) {
        e.preventDefault();
        boardList.search({ scope: $('#searchType').val(), text: $('#searchString').val() });
    });

    /* 표시 개수 변경 */
    $('#btnPage').on('click', function () {
        // setSize()는 현재 페이지를 유지하므로 범위 초과 방지를 위해 1로 초기화
        // (boardList.config.config === board.js 내부 boardConfig)
        boardList.config.config.page = 1;
        boardList.setSize($('#pageSize').val());
    });
});
</script>
```

---

## 3. 프론트: `info_view.jsp`

### `#searchForm` [변경]

```jsp
<form id="searchForm" method="POST">
    <input type="hidden" name="selScope"       id="selScope" value="<c:out value='${selScope}'/>">
    <input type="hidden" name="subSech"        id="subSech"  value="<c:out value='${subSech}'/>">
    <input type="hidden" name="size"           id="size"     value="<c:out value='${size}'/>">
    <input type="hidden" name="page"           id="page"     value="<c:out value='${page}'/>">
    <%-- [추가] --%>
    <input type="hidden" name="seCd"           id="seCd"     value="0002">
    <%-- [추가] --%>
    <input type="hidden" name="tab"            id="tab"      value="<c:out value='${tab}'/>">
    <input type="hidden" name="ntcnYardOrdrNo" id="ntcnYardOrdrNo" value="<c:out value='${ntcnYardOrdrNo}'/>">
</form>
```

---

## 4. 적용 전 확인

| 항목 | 확인 내용 |
|---|---|
| ▼▲ 사이 4줄 | 공통 `selectBoardListFile.do`에서 `MngNotificationVO`에 검색어·페이징 값을 넣는 줄을 복사하고 page=1, size=`INFO_FETCH_MAX`로만 변경 |
| 공통 서비스 | 같은 메서드에서 서비스 빈 이름·타입·호출 메서드명과 호출 후 후처리(fileId 암호화 등) 확인 |
| 반환 타입 | `List<EgovMap>`이면 목록 타입을 맞추고 제목은 `(String) item.get("ntcnYardSjNm")` |
| size 상한 | 공통 쪽에서 size를 제한하는지 (1000건 일괄 조회가 적용되는지) |
| 인터셉터 | `/kor/**` 인터셉터가 model에 값을 추가하면 신규 API 경로 2개를 `exclude-mapping` |
