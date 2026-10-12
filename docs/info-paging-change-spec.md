# 재정정보 탭 서버 페이징 — 최종 수정안

## 0. 수정 범위

| 구분 | 파일 | 수정 |
|---|---|---|
| JSP | `info.jsp` | **변경**: `#searchForm`에 `tab` 1줄, 스크립트 교체 |
| JSP | `info_view.jsp` | 없음 |
| Controller | `NotificationController.java` | **추가**: 탭 목록·건수 API 2개 + private 메서드 2개 |
| Controller | `info.do`, `info_view.do` 메서드 | 없음 |
| Service | 공통 목록 조회 서비스 | 없음 (기존 메서드 재사용) |
| Mapper / 공통 JS | 공통 Mapper, `board.js`, `paging.js` | 없음 |

---

## 1. JSP — `info.jsp`

### 1-1. `#searchForm` [추가 1줄]

```jsp
<form id="searchForm" method="POST">
    <input type="hidden" name="selScope" id="selScope" value="">
    <input type="hidden" name="subSech"  id="subSech"  value="">
    <input type="hidden" name="size"     id="size"     value="10">
    <input type="hidden" name="page"     id="page"     value="1">
    <input type="hidden" name="seCd"     id="seCd"     value="0002">
    <input type="hidden" name="tab"      id="tab"      value="biz">   <%-- [추가] --%>
    <input type="hidden" name="ntcnYardOrdrNo" id="ntcnYardOrdrNo" value="">
</form>
```
기존 `value` 출력 방식(`${size}` 등)은 그대로 두고 `tab` 줄만 추가합니다.

### 1-2. 탭 마크업 [변경 없음]

기존 `<nav class="link_tab ...">` / `<li data-tab="biz">` / `<li data-tab="etc">` 그대로 사용합니다.

### 1-3. 기존 스크립트 [주석처리]

`<script src="/js/cmn/paging.js"></script>` 아래의 기존 `(function(){ 'use strict'; ... })();` 블록 전체를 `<%-- --%>`로 감쌉니다.
(`loadAllPages`, `classify`, `renumber`, `loadBoard`, `runSearch`, `afterLoad`, `render`, `renderPaging`, `goPaging_PagingViewBiz/Etc`, `bindEvents`)

### 1-4. 신규 스크립트 [추가]

```html
<script>
var boardList;    // board.js boardOutputList()가 0건일 때 전역 boardList.empty() 호출
var viewLink;
var goPaging_PagingView = function (cPage) { boardList.setPage(cPage); };

jQuery(function () {
    var $tabs = $('.link_tab li[data-tab]');

    // 상세(info_view)에서 '목록'으로 돌아온 경우에만 마지막 탭 복원 (info_view 수정 없이 탭 유지)
    var savedTab = null;
    try {
        if (document.referrer.indexOf('/info_view.do') > -1) {
            savedTab = sessionStorage.getItem('infoTab');
        }
    } catch (e) {}
    setTab(savedTab === 'etc' ? 'etc' : 'biz');

    boardList = new getBoardData({
        url:      '/kor/notification/selectInfoList.do',     // 탭 목록 API
        urlCount: '/kor/notification/selectInfoListCnt.do',  // 탭 건수 API
        viewer:   '/kor/notification/info_view.do',
        complete: function (data) {
            $('#list').html(boardOutputList('textBoard', data.result));
        }
    });

    viewLink = function (id) { boardList.viewLink(id); };
    boardList.getList();

    /* 탭 클릭 : 서버에 해당 탭 1페이지 요청 */
    $tabs.on('click', 'a', function (e) {
        e.preventDefault();
        var tab = $(this).parent().data('tab');
        if (tab === $('#tab').val()) return;

        setTab(tab);
        $('#page').val(1);
        boardList.getList({ type: 'search' });   // 'search' → 탭 기준 건수도 다시 조회
    });

    /* 검색 : 현재 탭 안에서 검색 */
    $('#boardSearchForm').on('submit', function (e) {
        e.preventDefault();
        boardList.search({ scope: $('#searchType').val(), text: $('#searchString').val() });
    });

    /* 보기(표시 개수) */
    $('#btnPage').on('click', function () {
        boardList.config.config.page = 1;
        boardList.setSize($('#pageSize option:selected').val());
    });

    function setTab(tab) {
        $('#tab').val(tab);
        $tabs.removeClass('current').children('a').removeAttr('title');
        $tabs.filter('[data-tab="' + tab + '"]').addClass('current')
             .children('a').attr('title', '현재 페이지');
        try { sessionStorage.setItem('infoTab', tab); } catch (e) {}
    }
});
</script>
```

---

## 2. Controller — `NotificationController.java`

### 2-1. 필드 [추가]

공통 `/cmn/board/selectBoardListFile.do` 컨트롤러가 주입받는 **목록 조회 서비스와 같은 서비스**를 주입합니다 (이미 있으면 생략).

```java
@Autowired
private (공통 목록 서비스 타입) (공통 목록 서비스 변수);
```

### 2-2. 탭 목록·건수 API [추가]

```java
/** 재정정보 탭 목록 : 현재 페이지 분량만 반환 */
@RequestMapping(value = "/notification/selectInfoList.do")
public String selectInfoList(HttpServletRequest request, ModelMap model) throws Exception {
    int size = toInt(request.getParameter("size"), 10, 50);
    int page = toInt(request.getParameter("page"), 1, 100000);

    List<Map<String, Object>> tabList = selectInfoTabList(request);

    int from = Math.min((page - 1) * size, tabList.size());
    int to   = Math.min(from + size, tabList.size());

    model.addAttribute("result", new ArrayList<Map<String, Object>>(tabList.subList(from, to)));
    return "jsonView";
}

/** 재정정보 탭 건수 */
@RequestMapping(value = "/notification/selectInfoListCnt.do")
public String selectInfoListCnt(HttpServletRequest request, ModelMap model) throws Exception {
    model.addAttribute("result", selectInfoTabList(request).size());
    return "jsonView";
}

/** 공통 목록 조회와 같은 방식으로 재정정보 전체를 조회한 뒤 요청 탭의 글만 반환 */
private List<Map<String, Object>> selectInfoTabList(HttpServletRequest request) throws Exception {
    boolean bizTab = !"etc".equals(request.getParameter("tab"));

    // 공통 selectBoardListFile.do와 같은 params 구성 (page·size·seCd만 고정값)
    HashMap<String, Object> params = new HashMap<String, Object>();
    params.put("page", "1");                // 전체 조회
    params.put("size", "1000");
    params.put("seCd", "0002");             // 재정정보 고정
    params.put("selScope",        request.getParameter("selScope"));
    params.put("subSech",         request.getParameter("subSech"));
    params.put("viewType",        request.getParameter("viewType"));
    params.put("atchmnflKndSeCd", request.getParameter("atchmnflKndSeCd"));
    params.put("ntcnYardKndSeCd", request.getParameter("ntcnYardKndSeCd"));
    // ※ 공통 메서드가 위 값 외에 추가로 put 하는 값(가공값 등)이 있으면 똑같이 추가

    // 공통 selectBoardListFile.do와 같은 서비스·메서드 호출
    List<Map<String, Object>> allList = (공통 목록 서비스 변수).(공통 목록 메서드)(params);
    // ※ 공통 메서드가 호출 뒤 목록을 가공하면(fileList, fileId 등) 같은 코드를 여기에 추가

    List<Map<String, Object>> tabList = new ArrayList<Map<String, Object>>();
    for (Map<String, Object> row : allList) {
        Object title = row.get("ntcnYardSjNm");
        boolean isBiz = title != null && title.toString().contains("업무추진비");
        if (isBiz == bizTab) {
            tabList.add(row);
        }
    }
    return tabList;
}

private static int toInt(String v, int def, int max) {
    try {
        int n = Integer.parseInt(v);
        return (n < 1 || n > max) ? def : n;
    } catch (NumberFormatException e) {
        return def;
    }
}
```

import: `java.util.ArrayList`, `java.util.HashMap`, `java.util.List`, `java.util.Map`, `org.springframework.beans.factory.annotation.Autowired`

- 목록 타입 `Map<String, Object>`는 공통 메서드의 반환 타입으로 맞춥니다 (예: `List<EgovMap>`이면 `Map<String, Object>` 자리를 모두 `EgovMap`으로).
- 제목 키 `ntcnYardSjNm`은 `board.js`가 응답에서 `item.ntcnYardSjNm`으로 읽는 이름과 같습니다.

---

## 3. Service — 변경 없음

공통 목록 조회 서비스를 그대로 호출합니다. 비공개·삭제 등 공통 조회 조건(DB 90건 → 화면 68건)과 첨부파일·상단고정 정렬이 그대로 적용됩니다.

---

## 4. 적용 전 확인

| 항목 | 확인 방법 |
|---|---|
| params | 공통 `selectBoardListFile.do`의 `params.put(...)` 줄을 그대로 복사하고 page·size·seCd만 고정값으로 |
| 서비스·메서드·반환 타입 | 공통 `selectBoardListFile.do`의 서비스 호출 줄 그대로 |
| 호출 후 가공 | 공통 메서드에 있으면 같은 코드 추가 |
| `jsonView` | 공통 메서드의 return 방식과 동일하게 (`jsonView` / `@ResponseBody`) |

---

## 5. 동작 확인

| 시나리오 | 기대 결과 |
|---|---|
| 최초 진입 | 요청 2회(목록·건수), 업무추진비 탭, 총 건수 = 업무추진비 건수 |
| 기타 탭 클릭 | 요청 2회, 기타 건수, 1페이지 |
| 페이지 클릭 | 요청 1회 |
| 기타 탭 2페이지 → 상세 → 목록 | 기타 탭 2페이지로 복귀 |
| 메뉴로 새로 진입 | 업무추진비 탭 1페이지 |
| 탭 안에서 검색 | 현재 탭 결과만 |
| 공지사항 등 타 게시판 | 변화 없음 |
