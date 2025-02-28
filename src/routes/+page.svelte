<script>
    import { onDestroy } from "svelte";
    // 4. 역대 로또 번호 데이터 별도 파일에서 import
    import { officialLottoHistory } from "./officialData.js";

    /****************************************
     * ① 상태 변수들
     ****************************************/
    let isRunning = false; // 시뮬레이션 진행 여부
    let trialCount = 0; // 총 게임(시행) 횟수
    let results = {
        "1등": 0,
        "2등": 0,
        "3등": 0,
        "4등": 0,
        "5등": 0,
        꽝: 0,
    };

    let lastDraws = []; // 최근 당첨 기록 (최대 10회)
    let speed = 10; // 기본 배속
    let intervalId = null;

    // 로또 한 장당 가격
    const TICKET_PRICE = 1000;

    // 등수별 가상 당첨금
    const PRIZE_MONEY = {
        "1등": 2000000000, // 20억
        "2등": 70000000, // 7천만
        "3등": 1500000, // 150만
        "4등": 50000, // 5만
        "5등": 5000, // 5천
        꽝: 0,
    };

    // 1~5등 당첨 시 중단 체크박스
    let stopOn1st = false;
    let stopOn2nd = false;
    let stopOn3rd = false;
    let stopOn4th = false;
    let stopOn5th = false;

    // 번호 선택 관련
    let selectedNumbers = [];
    let playerTicket = [];

    // 역대 로또 번호 패널 토글
    let showOfficialHistory = false;

    /****************************************
     * ② 파생 상태(reactive declarations)
     ****************************************/
    $: totalCost = trialCount * TICKET_PRICE;
    $: totalPrize =
        results["1등"] * PRIZE_MONEY["1등"] +
        results["2등"] * PRIZE_MONEY["2등"] +
        results["3등"] * PRIZE_MONEY["3등"] +
        results["4등"] * PRIZE_MONEY["4등"] +
        results["5등"] * PRIZE_MONEY["5등"];

    $: roi = totalCost > 0 ? ((totalPrize / totalCost) * 100).toFixed(2) : 0;
    $: avgPrize = trialCount > 0 ? (totalPrize / trialCount).toFixed(2) : 0;

    // 게임 진행 중이면 번호 변경 불가
    $: canChangeNumbers = !isRunning;

    // 실시간 당첨 확률 (각 등수별 %)
    $: probabilities = {};
    $: {
        let total = trialCount > 0 ? trialCount : 1;
        for (const rank in results) {
            probabilities[rank] =
                ((results[rank] / total) * 100).toFixed(4) + "%";
        }
    }

    /****************************************
     * ③ 주요 함수들
     ****************************************/

    // (1) 번호 선택/해제
    function toggleNumber(num) {
        // 게임 진행 중이면 변경 불가
        if (!canChangeNumbers) return;

        if (selectedNumbers.includes(num)) {
            selectedNumbers = selectedNumbers.filter((n) => n !== num);
        } else {
            if (selectedNumbers.length < 6) {
                selectedNumbers = [...selectedNumbers, num].sort(
                    (a, b) => a - b,
                );
            } else {
                alert("이미 6개를 선택하셨습니다. 먼저 하나를 해제해주세요!");
            }
        }
    }

    // (2) 선택번호만 초기화
    function resetSelectedNumbers() {
        if (!canChangeNumbers) return;
        selectedNumbers = [];
    }

    // (3) "확정" 버튼: 반드시 눌러야 playerTicket에 반영
    function confirmTicket() {
        if (!canChangeNumbers) return;
        if (selectedNumbers.length !== 6) {
            alert("정확히 6개 번호를 선택해야 합니다!");
            return;
        }
        // 최종 확정
        playerTicket = [...selectedNumbers];
    }

    // (4) "자동 선택" 버튼: selectedNumbers만 변경, 확정은 confirmTicket으로
    function autoSelect() {
        if (!canChangeNumbers) return;
        selectedNumbers = generateRandomTicket();
        // playerTicket = [...selectedNumbers]; <-- 제거
    }

    // 랜덤 티켓
    function generateRandomTicket() {
        const nums = [];
        while (nums.length < 6) {
            const num = Math.floor(Math.random() * 45) + 1;
            if (!nums.includes(num)) nums.push(num);
        }
        return nums.sort((a, b) => a - b);
    }

    // (5) 시뮬레이션 시작
    //    - 1000배속, 10000배속이 실제로 빠르게 동작하도록
    //      1ms 주기로 speed번 반복 추첨
    function startSimulation() {
        if (playerTicket.length < 6) {
            alert("6개 번호를 확정해주세요!");
            return;
        }
        if (isRunning) return;

        isRunning = true;
        // 1ms 주기로 speed번 draw 실행
        intervalId = setInterval(() => {
            for (let i = 0; i < speed; i++) {
                simulateDraw();
            }
        }, 1);
    }

    // (6) 일시정지
    function pauseSimulation() {
        isRunning = false;
        if (intervalId) {
            clearInterval(intervalId);
            intervalId = null;
        }
    }

    // (7) 배속 조정
    //    - 이미 실행 중이면 setInterval 재설정
    function setSpeed(newSpeed) {
        speed = newSpeed;
        if (isRunning) {
            clearInterval(intervalId);
            intervalId = setInterval(() => {
                for (let i = 0; i < speed; i++) {
                    simulateDraw();
                }
            }, 1);
        }
    }

    // (8) 중단(초기화)
    function stopSimulation() {
        pauseSimulation();
        trialCount = 0;
        results = { "1등": 0, "2등": 0, "3등": 0, "4등": 0, "5등": 0, 꽝: 0 };
        lastDraws = [];
    }

    // (9) 1회 로또 추첨
    function simulateDraw() {
        trialCount += 1;
        let drawResult = checkDraw();

        results[drawResult.prize] += 1;

        lastDraws.unshift({
            round: trialCount,
            winningNumbers: drawResult.winningNumbers,
            bonus: drawResult.bonus,
            matchCount: drawResult.matchCount,
            prize: drawResult.prize,
        });
        if (lastDraws.length > 10) {
            lastDraws.pop();
        }

        // 당첨 시 중단 체크
        if (
            (stopOn1st && drawResult.prize === "1등") ||
            (stopOn2nd && drawResult.prize === "2등") ||
            (stopOn3rd && drawResult.prize === "3등") ||
            (stopOn4th && drawResult.prize === "4등") ||
            (stopOn5th && drawResult.prize === "5등")
        ) {
            pauseSimulation();
        }
    }

    // 추첨 결과 판정
    function checkDraw() {
        const numbers = Array.from({ length: 45 }, (_, i) => i + 1);
        shuffle(numbers);

        const winningNumbers = numbers.slice(0, 6).sort((a, b) => a - b);
        const bonus = numbers[6];

        const matchCount = playerTicket.filter((n) =>
            winningNumbers.includes(n),
        ).length;
        const bonusMatch = playerTicket.includes(bonus);

        let prize = "꽝";
        if (matchCount === 6) prize = "1등";
        else if (matchCount === 5 && bonusMatch) prize = "2등";
        else if (matchCount === 5) prize = "3등";
        else if (matchCount === 4) prize = "4등";
        else if (matchCount === 3) prize = "5등";

        return { prize, winningNumbers, bonus, matchCount, bonusMatch };
    }

    // Fisher-Yates 섞기
    function shuffle(array) {
        for (let i = array.length - 1; i > 0; i--) {
            const j = Math.floor(Math.random() * (i + 1));
            [array[i], array[j]] = [array[j], array[i]];
        }
    }

    // 역대 로또 번호 패널 열고/닫기
    function toggleOfficialHistory() {
        showOfficialHistory = !showOfficialHistory;
    }

    function formatKoreanCurrency(amount) {
        if (amount >= 100000000) {
            return (amount / 100000000).toFixed(2) + "억";
        } else if (amount >= 1000000) {
            return (amount / 1000000).toFixed(0) + "백만";
        } else if (amount >= 1000) {
            return (amount / 1000).toFixed(0) + "천";
        } else {
            return amount + "";
        }
    }

    // 언마운트 시
    onDestroy(() => {
        if (intervalId) clearInterval(intervalId);
    });
</script>

<div class="container">
    <h1>🤪 오물개 로또 시뮬레이션</h1>
    <p class="subtitle">번호 선택부터 시뮬레이션까지, 재미로만 즐겨야함!</p>

    <div class="row">
        <!-- 왼쪽: 시뮬레이션 영역 -->
        <div class="col simulation-col">
            <!-- [1] 번호 선택 영역 -->
            <div class="select-area card">
                <h2 class="card-title">✅ 번호 선택</h2>
                <p class="desc">
                    아래 번호를 클릭하여 <strong>최대 6개</strong>를 선택하세요.
                </p>

                <div class="number-grid">
                    {#each Array(7) as _, rowIndex}
                        <div class="row">
                            {#each Array(7) as _, colIndex}
                                {#if rowIndex * 7 + colIndex < 45}
                                    <button
                                        class:selected={selectedNumbers.includes(
                                            rowIndex * 7 + colIndex + 1,
                                        )}
                                        disabled={!canChangeNumbers}
                                        on:click={() =>
                                            toggleNumber(
                                                rowIndex * 7 + colIndex + 1,
                                            )}
                                    >
                                        {rowIndex * 7 + colIndex + 1}
                                    </button>
                                {/if}
                            {/each}
                        </div>
                    {/each}
                </div>

                <!-- 선택된 번호 영역 (새로운 구조) -->
                <div class="selected-numbers">
                    <h4 class="card-subtitle">선택된 번호</h4>
                    {#if selectedNumbers.length > 0}
                        <div class="number-badges">
                            {#each selectedNumbers as num}
                                <span class="badge">{num}</span>
                            {/each}
                        </div>
                    {:else}
                        <p class="placeholder">
                            아직 번호가 선택되지 않았습니다.
                        </p>
                    {/if}
                </div>

                <div class="controls">
                    <button
                        class="btn-confirm"
                        on:click={confirmTicket}
                        disabled={!canChangeNumbers}
                    >
                        확정
                    </button>
                    <button
                        class="btn-auto"
                        on:click={autoSelect}
                        disabled={!canChangeNumbers}
                    >
                        자동 선택
                    </button>
                    <button
                        class="btn-reset"
                        on:click={resetSelectedNumbers}
                        disabled={!canChangeNumbers}
                    >
                        선택 해제
                    </button>
                </div>
            </div>

            <!-- [2] 내 로또 번호 & 시뮬레이션 컨트롤 영역 -->
            <div class="my-ticket-area card">
                <h2 class="card-title">🕹️ 내 로또 번호 &amp; 컨트롤</h2>
                <p class="ticket">
                    <strong>내 로또 번호 (확정):</strong>
                    {#if playerTicket.length === 6}
                        {playerTicket.join(", ")}
                    {:else}
                        (아직 미확정)
                    {/if}
                </p>
                <div class="controls">
                    <button class="btn-start" on:click={startSimulation}>
                        시작
                    </button>
                    <button class="btn-pause" on:click={pauseSimulation}>
                        일시정지
                    </button>
                    <button class="btn-stop" on:click={stopSimulation}>
                        중단(초기화)
                    </button>
                </div>
                <div class="controls speed-controls">
                    <button
                        class:selected={speed == 10}
                        on:click={() => setSpeed(10)}
                    >
                        10배속
                    </button>
                    <button
                        class:selected={speed == 100}
                        on:click={() => setSpeed(100)}
                    >
                        100배속
                    </button>
                    <button
                        class:selected={speed == 1000}
                        on:click={() => setSpeed(1000)}
                    >
                        1000배속
                    </button>
                </div>
                <div class="checklist">
                    <h4 class="card-subtitle">일시정지 조건</h4>
                    <p class="desc">체크된 등수 당첨시 일시정지 됩니다.</p>
                    <div class="checkItem">
                        <label
                            ><input type="checkbox" bind:checked={stopOn1st} /> 1등</label
                        >
                        <label
                            ><input type="checkbox" bind:checked={stopOn2nd} /> 2등</label
                        >
                        <label
                            ><input type="checkbox" bind:checked={stopOn3rd} /> 3등</label
                        >
                        <label
                            ><input type="checkbox" bind:checked={stopOn4th} /> 4등</label
                        >
                        <label
                            ><input type="checkbox" bind:checked={stopOn5th} /> 5등</label
                        >
                    </div>
                </div>
            </div>

            <!-- [3] 시뮬레이션 결과 영역 (구조 개선) -->
            <div class="simulation-results card">
                <h3 class="card-title">💰 시뮬레이션 결과</h3>
                <div class="results-summary">
                    <div class="result-item">
                        <span class="label">시행 횟수</span>
                        <span class="value"
                            >{formatKoreanCurrency(trialCount)}회</span
                        >
                    </div>
                    <div class="result-item">
                        <span class="label">총 비용</span>
                        <span class="value"
                            >{formatKoreanCurrency(totalCost)}원</span
                        >
                    </div>
                    <div class="result-item">
                        <span class="label">당첨 수익</span>
                        <span class="value"
                            >{formatKoreanCurrency(totalPrize)}원</span
                        >
                    </div>
                    <div class="result-item">
                        <span class="label">수익률 (ROI)</span>
                        <span class="value">{roi}%</span>
                    </div>
                    <!-- <div class="result-item">
                        <span class="label">평균 당첨금 (게임당)</span>
                        <span class="value">{avgPrize}원</span>
                    </div> -->
                </div>
                <div class="results-details">
                    <h4>당첨 상세 결과</h4>
                    <ul>
                        <li>
                            <strong>1등 (20억) :</strong>
                            {results["1등"]}
                            <span class="probability"
                                >({probabilities["1등"]})</span
                            >
                        </li>
                        <li>
                            <strong>2등 (7천만) :</strong>
                            {results["2등"]}
                            <span class="probability"
                                >({probabilities["2등"]})</span
                            >
                        </li>
                        <li>
                            <strong>3등 (150만) :</strong>
                            {results["3등"]}
                            <span class="probability"
                                >({probabilities["3등"]})</span
                            >
                        </li>
                        <li>
                            <strong>4등 (5만) :</strong>
                            {results["4등"]}
                            <span class="probability"
                                >({probabilities["4등"]})</span
                            >
                        </li>
                        <li>
                            <strong>5등 (5천) :</strong>
                            {results["5등"]}
                            <span class="probability"
                                >({probabilities["5등"]})</span
                            >
                        </li>
                        <li>
                            <strong>꽝:</strong>
                            {results["꽝"]}
                            <span class="probability"
                                >({probabilities["꽝"]})</span
                            >
                        </li>
                    </ul>
                </div>
            </div>
        </div>

        <!-- 오른쪽: 역대 로또 번호 패널 (새로운 카드 형식) -->
        <div class="slide-panel {showOfficialHistory ? 'show' : ''}">
            <div class="historical-draws card">
                <h2 class="card-title">역대 로또 번호</h2>
                <p class="date-note">2025-02-22 기준</p>
                <div class="draw-list">
                    {#each officialLottoHistory as history}
                        <div class="draw-item">
                            <div class="draw-header">
                                <span class="draw-round">#{history.round}</span>
                                <span class="draw-prize">
                                    {formatKoreanCurrency(history.firstPrize)} ({history.firstWinners}명)
                                </span>
                            </div>
                            <div class="draw-numbers">
                                <span class="numbers"
                                    >{history.numbers.join(", ")}</span
                                >
                                <span class="bonus">+ {history.bonus}</span>
                            </div>
                        </div>
                    {/each}
                </div>
            </div>
        </div>

        <!-- "역대 로또 번호 보기" 토글 버튼 -->
        <button class="btn-history toggle-btn" on:click={toggleOfficialHistory}>
            {#if showOfficialHistory}
                역대 로또 번호 닫기
            {:else}
                역대 로또 번호 보기
            {/if}
        </button>
    </div>

    <div class="footer-note">
        <p>
            ※ 이 시뮬레이터는 재미를 위한 것이며, 실제 복권 당첨금이나
            수익률과는 무관합니다.
        </p>
        <a class="link" href="https://blog.omoolgae.site/">뭐라도 만드는 오물개 블로그 방문하기</a>
    </div>
</div>

<!-- 수정한 영역 관련 CSS -->
<style>
    /* 컨테이너 및 기본 폰트 */
    .container {
        max-width: 800px;
        margin: 2rem auto;
        padding: 1.5rem;
        font-family: "Noto Sans KR", "Helvetica Neue", Helvetica, Arial,
            sans-serif;
        border: 1px solid #ddd; 
        border-radius: 8px;
        background-color: #fafafa;
    }
    .link{
        text-decoration: underline;
        color: #7f7f7f;
    }
    /* 카드 기본 스타일 */
    .card {
        background-color: #fff;
        border: 1px solid #e0e0e0;
        border-radius: 8px;
        padding: 1.5rem;
        margin-bottom: 2rem;
        box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
    }
    .card-title {
        font-size: 1.5rem;
        font-weight: 600;
        margin-bottom: 1rem;
    }
    .card-subtitle {
        font-size: 1.25rem;
        font-weight: 500;
        margin-bottom: 0.5rem;
    }

    .number-grid {
        margin: 1rem auto;
        max-width: 400px;
    }
    .number-grid .row {
        display: flex;
        justify-content: flex-start;
        margin-bottom: 8px;
    }
    .number-grid button {
        width: 60px;
        margin: 4px;
        padding: 8px;

        border-radius: 4px;
        background-color: #a9a9a9;
        cursor: pointer;
        transition: background-color 0.3s ease;
    }
    .number-grid button:hover {
        background-color: #b7b7b7;
    }
    .number-grid button.selected {
        background-color: #ffaf45;
        font-weight: bold;
        color: #fff;
    }

    /* 선택된 번호 영역 */
    .selected-numbers {
        margin: 1rem 0 1.5rem;
    }
    .selected-numbers .number-badges {
        display: flex;
        flex-wrap: wrap;
        gap: 0.5rem;
    }
    .badge {
        background-color: #ffaf45;
        color: #fff;
        padding: 0.5rem 0.75rem;
        border-radius: 4px;
        font-weight: 600;
        font-size: 1rem;
    }
    .selected-numbers .placeholder {
        color: #888;
        font-style: italic;
    }

    /* 시뮬레이션 영역 */
    .my-ticket-area {
        margin-bottom: 2rem;
    }
    .controls {
        margin: 1rem 0;
    }
    .speed-controls {
        margin-top: 1rem;
    }
    .speed-controls button {
        border-radius: 4px;
        background-color: #a9a9a9;
        cursor: pointer;
        padding: 0.6rem 1rem;
        margin-right: 0.5rem;
        transition: background-color 0.3s ease;
    }
    .speed-controls button.selected {
        background-color: #ffaf45;
        font-weight: bold;
        color: #fff;
    }

    /* 시뮬레이션 결과 영역 */
    .simulation-results .results-summary {
        display: grid;
        grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
        gap: 1rem;
        margin-bottom: 1.5rem;
    }
    .result-item {
        display: flex;
        justify-content: space-between;
        font-size: 1rem;
    }
    .result-item .label {
        font-weight: 500;
        color: #333;
    }
    .result-item .value {
        font-weight: 600;
    }
    .results-details h4 {
        font-size: 1.25rem;
        font-weight: 500;
        margin-bottom: 0.75rem;
    }
    .results-details ul {
        list-style: none;
        padding: 0;
    }
    .results-details li {
        font-size: 1rem;
        padding: 0.5rem 0;
        border-bottom: 1px dashed #ddd;
    }
    .results-details li:last-child {
        border-bottom: none;
    }
    .results-details .probability {
        color: #888;
        margin-left: 0.5rem;
    }
    /* 버튼 스타일 */
    button {
        cursor: pointer;
        border: none;
        border-radius: 4px;
        padding: 0.6rem 1rem;
        margin: 0.3rem;
        color: #fff;
        font-size: 1rem;
        transition:
            background-color 0.3s ease,
            transform 0.2s ease;
    }
    button:hover {
        transform: scale(1.03);
    }
    .btn-start {
        background-color: #28a745;
    }
    .btn-pause {
        background-color: #ffc107;
        color: #000;
    }
    .btn-stop {
        background-color: #dc3545;
    }
    .btn-confirm {
        background-color: #17a2b8;
    }
    .btn-auto {
        background-color: #6f42c1;
    }
    .btn-reset {
        background-color: #343a40;
    }
    .btn-history {
        background-color: #007bff;
        margin-bottom: 1rem;
    }

    /* 역대 로또 번호 (역사적 추첨 결과) */
    .historical-draws .date-note {
        font-size: 0.9rem;
        color: #666;
        margin-bottom: 1rem;
    }
    .historical-draws .draw-list {
        display: flex;
        flex-direction: column;
        gap: 1rem;
    }
    .draw-item {
        padding: 1rem;
        border: 1px solid #eaeaea;
        border-radius: 6px;
        transition: background-color 0.3s;
    }
    .draw-item:hover {
        background-color: #f9f9f9;
    }
    .draw-header {
        display: flex;
        justify-content: space-between;
        font-size: 1rem;
        margin-bottom: 0.5rem;
        font-weight: 500;
    }
    .draw-numbers {
        font-size: 1rem;
        color: #333;
    }
    .draw-numbers .bonus {
        margin-left: 0.5rem;
        color: #ff5722;
        font-weight: 600;
    }

    /* 슬라이드 패널 */
    .slide-panel {
        position: fixed;
        top: 0;
        right: 0;
        width: 350px;
        height: 100%;
        background-color: #fff;
        border-left: 1px solid #ddd;
        transform: translateX(100%);
        transition: transform 0.3s ease;
        overflow-y: auto;
        /* padding: 2rem; */
        box-shadow: -2px 0 10px rgba(0, 0, 0, 0.1);
    }
    .slide-panel.show {
        transform: translateX(0);
    }
    /* 토글 버튼 */
    .toggle-btn {
        position: fixed;
        bottom: 100px;
        right: 20px;
        z-index: 999;
        padding: 0.6rem 1rem;
        border-radius: 4px;
        background-color: #007bff;
        color: #fff;
        font-size: 1rem;
        transition: background-color 0.3s ease;
    }
    .toggle-btn:hover {
        background-color: #0056b3;
    }

    /* 푸터 노트 */
    .footer-note {
        font-size: 0.9rem;
        color: gray;
        margin-top: 2rem;
        text-align: center;
    }
</style>
