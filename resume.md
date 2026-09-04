
k: WebSquare GridView 동적 멀티라인 CheckComboBox 구현

아래 요구사항에 맞춰 WebSquare XML 파일을 생성하고 프로젝트에 반영해 줘.

#### [요구사항]
1. **계층형 동적 바인딩:** `workType`(작업구분) 선택에 따라 `workTarget`(작업대상)의 드롭다운 목록이 행(Row)별로 독립 동작 (`setCellItemList`).
2. **다중 선택 지원:** `inputType="checkcombobox"` 및 쉼표(`,`) 구분자 처리.
3. **체크 항목 우선 정렬:** 사용자가 체크한 항목은 드롭다운 목록 최상단으로 재정렬.
4. **멀티라인 표시:** 포커스 아웃(읽기 모드) 시 `customFormatter`와 CSS(`white-space: pre-wrap`)를 통해 설비ID와 명칭이 줄바꿈(`\n`)되어 출력되도록 구현.
5. **초기 조회 연동:** `scwin.fn_searchCallback`을 통해 데이터 조회 시 각 행의 상태 복원.

---

#### [생성할 파일: `src/main/webapp/ui/work/WorkListGrid.xml`]

```xml
<?xml version="1.0" encoding="UTF-8"?>
<html xmlns="http://www.w3.org/1999/xhtml"
    xmlns:ev="http://www.w3.org/2001/xml-events"
    xmlns:w2="http://www.inswave.com/websquare"
    xmlns:xf="http://www.w3.org/2002/xforms">
    <head>
        <w2:type>COMPONENT</w2:type>
        <w2:buildDate/>
        <xf:model>
            <w2:dataCollection baseNode="map">
                <!-- 그리드 메인 DataList -->
                <w2:dataList id="dlt_grid" baseNode="list" repeatNode="map" saveRemovedData="true">
                    <w2:columnInfo>
                        <w2:column id="workType" name="작업구분" dataType="text"></w2:column>
                        <w2:column id="workTarget" name="작업대상" dataType="text"></w2:column>
                    </w2:columnInfo>
                </w2:dataList>

                <!-- 작업구분 공통코드 DataList -->
                <w2:dataList id="dlt_workType" baseNode="list" repeatNode="map">
                    <w2:columnInfo>
                        <w2:column id="code" name="코드" dataType="text"></w2:column>
                        <w2:column id="name" name="명칭" dataType="text"></w2:column>
                    </w2:columnInfo>
                    <w2:data xmlns="">
                        <w2:row>
                            <code>EQ</code>
                            <name>설비</name>
                        </w2:row>
                        <w2:row>
                            <code>MAT</code>
                            <name>자재입출고</name>
                        </w2:row>
                    </w2:data>
                </w2:dataList>
            </w2:dataCollection>

            <!-- 검색 Submission -->
            <xf:submission id="sbm_search" ref="" target="data:json,dlt_grid" action="/api/work/list" method="post" mediatype="application/json"
                encoding="UTF-8" instance="" replace="" errorHandler="" customHandler="" mode="asynchronous" processMsg="" ev:submitdone="scwin.sbm_search_submitdone">
            </xf:submission>
        </xf:model>

        <style type="text/css"><![CDATA[
            /* 그리드 멀티라인 셀 줄바꿈 스타일 */
            .multiline_cell {
                white-space: pre-wrap !important;
                line-height: 1.4 !important;
                padding: 6px 4px !important;
            }
        ]]></style>

        <script type="javascript" lazy="false"><![CDATA[
            /**
             * 상위 작업구분별 작업대상 코드 마스터 데이터
             */
            scwin.targetCodeMaster = {
                "EQ": [
                    { id: "EQ_A01", name: "압출 성형기 1호" },
                    { id: "EQ_B01", name: "고속 정밀 냉각기" },
                    { id: "EQ_C01", name: "자동 포장기 3호" },
                    { id: "EQ_D01", name: "원료 배합기 2호" }
                ],
                "MAT": [
                    { id: "IN_NORMAL", name: "정상입고" },
                    { id: "IN_URGENT", name: "긴급입고" },
                    { id: "OUT_NORMAL", name: "정상출고" },
                    { id: "OUT_DISCARD", name: "폐기출고" }
                ]
            };

            scwin.onpageload = function() {
                // 테스트용 초기 샘플 데이터 세팅
                var sampleData = [
                    { "workType": "EQ", "workTarget": "EQ_A01,EQ_C01" },
                    { "workType": "MAT", "workTarget": "IN_NORMAL" },
                    { "workType": "EQ", "workTarget": "EQ_B01" }
                ];
                dlt_grid.setJSON(sampleData);
                scwin.fn_initGridRender();
            };

            /**
             * 조회 완료 후 전체 행 동적 드롭다운 렌더링
             */
            scwin.sbm_search_submitdone = function(e) {
                scwin.fn_initGridRender();
            };

            scwin.fn_initGridRender = function() {
                var rowCount = dlt_grid.getRowCount();
                for (var i = 0; i < rowCount; i++) {
                    var workType = dlt_grid.getCellData(i, "workType");
                    var workTarget = dlt_grid.getCellData(i, "workTarget");
                    if (workType) {
                        scwin.refreshTargetItemList(i, workType, workTarget);
                    }
                }
            };

            /**
             * 그리드 셀 변경 이벤트 (onviewchange)
             */
            scwin.grd_workList_onviewchange = function(info) {
                var rowIndex = info.rowIndex;
                var colId = info.colId;
                var newValue = info.newValue;

                // 1. 작업구분 변경 시 -> 선택값 초기화 및 신규 목록 세팅
                if (colId === "workType") {
                    var targetColIndex = grd_workList.getColumnIndex("workTarget");
                    grd_workList.setCellData(rowIndex, targetColIndex, "");
                    scwin.refreshTargetItemList(rowIndex, newValue, "");
                }

                // 2. 작업대상 체크 변경 시 -> 체크된 항목 상단 재정렬
                if (colId === "workTarget") {
                    var currentWorkType = dlt_grid.getCellData(rowIndex, "workType");
                    scwin.refreshTargetItemList(rowIndex, currentWorkType, newValue);
                }
            };

            /**
             * 체크된 항목을 상단으로 정렬하여 setCellItemList 실행
             */
            scwin.refreshTargetItemList = function(rowIndex, workType, selectedValuesStr) {
                var targetColIndex = grd_workList.getColumnIndex("workTarget");
                var masterList = scwin.targetCodeMaster[workType] || [];

                if (masterList.length === 0) {
                    grd_workList.setCellItemList(rowIndex, targetColIndex, []);
                    return;
                }

                var selectedArr = selectedValuesStr ? selectedValuesStr.split(",") : [];
                var checkedItems = [];
                var uncheckedItems = [];

                masterList.forEach(function(item) {
                    var itemObj = {
                        label: item.id + "\n(" + item.name + ")",
                        value: item.id
                    };
                    if (selectedArr.indexOf(item.id) > -1) {
                        checkedItems.push(itemObj);
                    } else {
                        uncheckedItems.push(itemObj);
                    }
                });

                var sortedList = checkedItems.concat(uncheckedItems);
                grd_workList.setCellItemList(rowIndex, targetColIndex, sortedList);
            };

            /**
             * 포커스 아웃(읽기 모드) 커스텀 포맷터
             */
            scwin.fn_formatWorkTarget = function(value, formattedValue, rowIndex, colIndex) {
                if (!value) return "";

                var workType = dlt_grid.getCellData(rowIndex, "workType");
                var masterList = scwin.targetCodeMaster[workType] || [];
                var selectedValues = value.split(",");
                var resultTextArr = [];

                selectedValues.forEach(function(val) {
                    var found = masterList.find(function(item) {
                        return item.id === val;
                    });
                    if (found) {
                        resultTextArr.push(found.id + "\n" + found.name);
                    } else {
                        resultTextArr.push(val);
                    }
                });

                return resultTextArr.join("\n----------------\n");
            };
        ]]></script>
    </head>
    <body ev:onpageload="scwin.onpageload">
        <w2:gridView
            id="grd_workList"
            dataList="data:dlt_grid"
            autoRowHeight="true"
            ev:onviewchange="scwin.grd_workList_onviewchange"
            style="width:100%; height:350px;"
            visibleRowNum="10">
            <w2:header>
                <w2:row>
                    <w2:column id="col_type" value="작업구분" width="130"/>
                    <w2:column id="col_target" value="작업대상" width="250"/>
                </w2:row>
            </w2:header>
            <w2:gBody>
                <w2:row>
                    <w2:column
                        id="workType"
                        inputType="select"
                        chooseOption="true"
                        chooseOptionLabel="- 선택 -">
                        <w2:choices>
                            <w2:itemset nodeset="data:dlt_workType">
                                <w2:label ref="name"/>
                                <w2:value ref="code"/>
                            </w2:itemset>
                        </w2:choices>
                    </w2:column>
                    <w2:column
                        id="workTarget"
                        inputType="checkcombobox"
                        separator=","
                        allOption="true"
                        allOptionLabel="전체선택"
                        tooltipDisplay="true"
                        tooltipFormatter="scwin.fn_formatWorkTarget"
                        customFormatter="scwin.fn_formatWorkTarget"
                        class="multiline_cell"/>
                </w2:row>
            </w2:gBody>
        </w2:gridView>
    </body>
</html>