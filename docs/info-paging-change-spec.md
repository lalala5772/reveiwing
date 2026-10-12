# 재정정보 탭 서버 페이징 — 최종 수정안

## 0. 수정 범위

| 구분 | 파일 | 수정 |
|---|---|---|
| JSP | `info.jsp` | **변경**: `#searchForm`에 `tab` 1줄, 스크립트 교체 |
| JSP | `info_view.jsp` | 없음 |
| Controller | `NotificationController.java` | **추가**: 탭 목록·건수 API 2개 + private 메서드 2개 |
| Controller | `info.do`, `info_view.do` 메서드 | 없음 |
| Service | `MngNotificationService` | 없음 (기존 목록 조회 메서드 재사용) |
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

### 2-1. 필드 [추가] (이미 주입되어 있으면 생략)

```java
@Autowired
private MngNotificationService mngNotificationService;
```

### 2-2. 탭 목록·건수 API [추가]

```java
/** 재정정보 탭 목록 : 현재 페이지 분량만 반환 */
@RequestMapping(value = "/notification/selectInfoList.do")
public String selectInfoList(HttpServletRequest request, ModelMap model) throws Exception {
    int size = toInt(request.getParameter("size"), 10, 50);
    int page = toInt(request.getParameter("page"), 1, 100000);

    List<MngNotificationVO> tabList = selectInfoTabList(request);

    int from = Math.min((page - 1) * size, tabList.size());
    int to   = Math.min(from + size, tabList.size());

    model.addAttribute("result", new ArrayList<MngNotificationVO>(tabList.subList(from, to)));
    return "jsonView";
}

/** 재정정보 탭 건수 */
@RequestMapping(value = "/notification/selectInfoListCnt.do")
public String selectInfoListCnt(HttpServletRequest request, ModelMap model) throws Exception {
    model.addAttribute("result", selectInfoTabList(request).size());
    return "jsonView";
}

/** 재정정보 전체를 기존 서비스로 조회한 뒤 요청 탭에 해당하는 글만 반환 */
private List<MngNotificationVO> selectInfoTabList(HttpServletRequest request) throws Exception {
    boolean bizTab = !"etc".equals(request.getParameter("tab"));

    MngNotificationVO searchVO = new MngNotificationVO();
    searchVO.setNtcnYardTySeCd("0002");                       // 재정정보 고정
    // ▼ 공통 목록 API(/cmn/board/selectBoardListFile.do)와 같은 setter 사용 (이름 확인)
    searchVO.setSelScope(request.getParameter("selScope"));
    searchVO.setSubSech(request.getParameter("subSech"));
    searchVO.setPage("1");                                    // 전체 조회
    searchVO.setSize("1000");
    // ▲

    // 공통 목록 API가 호출하는 것과 같은 메서드 (메서드명 확인)
    List<MngNotificationVO> allList = mngNotificationService.selectBoardListFile(searchVO);

    List<MngNotificationVO> tabList = new ArrayList<MngNotificationVO>();
    for (MngNotificationVO vo : allList) {
        String title = vo.getNtcnYardSjNm();
        boolean isBiz = title != null && title.contains("업무추진비");
        if (isBiz == bizTab) {
            tabList.add(vo);
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

import: `java.util.ArrayList`, `java.util.List`, `org.springframework.beans.factory.annotation.Autowired`

---

## 3. Service — 변경 없음

`MngNotificationService`의 기존 목록 조회 메서드를 그대로 호출합니다.
첨부파일(`fileList`, 암호화 `fileId`)·상단고정 정렬이 공통 API와 동일하게 나옵니다.

---

## 4. 적용 전 확인 (코드에서 이름만 맞추면 되는 부분)

| 항목 | 확인 방법 |
|---|---|
| `▼▲` 4줄 setter 이름·타입 | 공통 `selectBoardListFile.do`에서 VO에 값을 넣는 코드와 동일하게 (int 타입이면 `1`, `1000`) |
| 서비스 메서드명 | 공통 `selectBoardListFile.do`가 호출하는 `MngNotificationService` 메서드 |
| 호출 후 후처리 | 공통 API가 서비스 호출 뒤 추가 가공(fileId 등)을 하면 동일하게 |
| `jsonView` | `dispatcher-servlet.xml`에 빈 존재 여부 (공통 API가 `@ResponseBody`면 그 방식으로) |
| size 상한 | 공통 쪽에서 size를 제한하면 그 상한이 재정정보 전체 건수보다 큰지 |

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
