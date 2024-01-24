<template>
    <div id="home">
        <div class="info">
            <p class="title">Pymills</p>
            <!-- Buttons for game type selection -->
            <div
                v-if="!gameType"
                class="game-type-chooser"
            >
                <div class="game-type-cards">
                    <button
                        v-for="game in EGameType"
                        :key="game"
                        @click="handleGameCardClick(game)"
                    >
                        {{ game }}
                    </button>
                </div>
            </div>

            <!-- Buttons for difficulty selection -->
            <div
                v-else-if="showDifficutlyChooser"
                class="difficulty-chooser"
            >
                <div
                    class="difficulty-cards"
                    v-if="showWhiteDifficultyChooser"
                >
                    <button
                        v-for="diff in EDifficulty"
                        :key="diff"
                        @click="handleDifficultyClick('white', diff)"
                    >
                        {{ diff }}
                    </button>
                </div>

                <div
                    class="difficulty-cards"
                    v-if="!showWhiteDifficultyChooser && showBlackDifficultyChooser"
                >
                    <button
                        v-for="diff in EDifficulty"
                        :key="diff"
                        @click="handleDifficultyClick('black', diff)"
                    >
                        {{ diff }}
                    </button>
                </div>
            </div>

            <!-- Display other info -->
            <div
                v-if="gameType"
                class="unplaced-stones"
            >
                <div
                    class="tex"
                    v-if="gameState.unplacedPieces.white > 0"
                >
                    <p>Nepostavljene bele: {{ gameState.unplacedPieces.white }}</p>
                </div>
                <div
                    class="tex"
                    v-if="gameState.unplacedPieces.black > 0"
                >
                    <p>Nepostavljene crne: {{ gameState.unplacedPieces.black }}</p>
                </div>
                <div
                    class="tex"
                    v-if="gameState.unplacedPieces.black == 0 && gameState.unplacedPieces.white == 0"
                >
                    <p>Faza kretanja je u toku</p>
                </div>
                <div class="tex">
                    <p
                        v-if="gameState.player == 'white'"
                        class="honk"
                    >
                        Beli je na potezu
                    </p>
                    <p
                        v-if="gameState.player != 'white'"
                        class="nig"
                    >
                        Crni je na potezu
                    </p>
                </div>
            </div>
            <div
                class="lastEval"
                v-if="lastEval != null"
            >
                <p>Evaluacija poteza:</p>
                <p>{{ lastEval }}</p>
            </div>
            <div
                class="lastEval"
                v-if="madeMove !== null && gameType"
            >
                <p v-if="madeMove[4] === null">{{ madeMove[2] ? "Beli" : "Crni" }} igrač je odigrao {{ madeMove[1] }}</p>
                <p v-else-if="madeMove[4] !== null">{{ madeMove[2] ? "Beli" : "Crni" }} igrač je odigrao {{ madeMove[4] }} na {{ madeMove[1] }}</p>
                <p v-if="madeMove[5] !== null">Uklonjena je figura sa {{ madeMove[5] }}</p>
            </div>
        </div>
        <div
            class="mica-container"
            v-bind:class="{ unclickable: !gameType }"
        >
            <div
                class="mica"
                ref="micaRef"
                v-resize="calculateGap"
            >
                <div
                    class="point"
                    v-for="point in calculatedPoints"
                    :key="`point-${point.x}-${point.y}`"
                    :class="{
                        nearby: nearbyFreePoints.includes(point.point),
                    }"
                    :style="{
                        left: point.x * micaGap + 'px',
                        top: point.y * micaGap + 'px',
                    }"
                    @click="handlePointClick(point.point)"
                >
                    <div class="point-inner"></div>
                </div>

                <div
                    class="connection"
                    v-for="connection in calculatedConnections"
                    :key="`connection-${connection.x1}-${connection.y1}-${connection.x2}-${connection.y2}`"
                    :style="{
                        left: connection.x1 * micaGap + 'px',
                        top: connection.y1 * micaGap + 'px',
                        width: (connection.x2 - connection.x1) * micaGap + 'px',
                        height: (connection.y2 - connection.y1) * micaGap + 'px',
                    }"
                ></div>

                <div
                    v-for="figure in calculatedFigures"
                    class="figure absolutely-positioned"
                    :class="{
                        selected: figure.point === selectedFigure,
                        black: figure.color === 'black',
                        white: figure.color === 'white',
                        canRemove: canRemoveOpponentsPiece && opponentsStones.includes(figure.point),
                        isInFormedMill: formedMillStones.includes(figure.point) && !(canRemoveOpponentsPiece && opponentsStones.includes(figure.point)),
                    }"
                    :style="{
                        left: figure.x * micaGap + 'px',
                        top: figure.y * micaGap + 'px',
                    }"
                    :key="`figure-${figure.x}-${figure.y}`"
                    v-click-outside="() => selectFigure(null)"
                    @click.stop="selectFigure(figure.point)"
                ></div>
            </div>
        </div>
    </div>
</template>

<script setup lang="ts">
    import { ref, onMounted, computed, watch } from "vue";
    import Client, { EDifficulty, type IGameState, type IMapObject, type ITakenPoint, type IPoint, type TPlayer } from "@/Client";
    import { vResize, vClickOutside } from "../vueDirectives";
    import axios from "axios";

    interface ICalculatedFigure {
        x: number;
        y: number;
        color: "black" | "white";
        point: ITakenPoint;
    }

    interface ICalculatedPoint {
        point: string;
        x: number;
        y: number;
    }

    interface ICalculatedConnection {
        x1: number;
        y1: number;
        x2: number;
        y2: number;
    }

    interface IPlayerData {
        name: string;
        color: TPlayer;
        timeout?: number;
        difficulty: EDifficulty | null;
        depth: number;
    }

    enum EGameType {
        PlayerVsPlayer = "Igrač vs Igrač",
        PlayerVsComputer = "Igrač vs Kompjuter",
        ComputerVsComputer = "Kompjuter vs Kompjuter",
    }

    const gameState = ref<IGameState>({
        player: "white",
        unplacedPieces: {
            black: 9,
            white: 9,
        },
        occupiedPoints: [],
    });

    const madeMove = ref<any[]>([]);

    const gameType = ref<EGameType | null>(null);

    const loading = ref<boolean>(false);

    const mapsData = ref<Array<IMapObject> | null>(null);
    const selectedMap = ref<IMapObject | null>(null);
    const micaSize = ref<number | null>(null);

    const micaRef = ref<HTMLElement | null>(null);
    const micaGap = ref<number>(0);

    const selectedFigure = ref<ITakenPoint | null>(null);

    const canRemoveOpponentsPiece = ref<boolean>(false);

    const lastEval = ref<number | null>(null);

    const whitePlayer = ref<IPlayerData>({
        name: "White player",
        color: "white",
        timeout: 30,
        difficulty: null,
        depth: 5,
    });

    const blackPlayer = ref<IPlayerData>({
        name: "Black player",
        color: "black",
        timeout: 30,
        difficulty: null,
        depth: 5,
    });

    const showOverlay = computed(() => {
        if (!gameType.value) return true;

        switch (gameType.value) {
            case EGameType.PlayerVsPlayer:
                if (loading.value) return true;
                break;
            case EGameType.PlayerVsComputer:
                if (loading.value) return true;
                if (showBlackDifficultyChooser.value) return true;
                break;
            case EGameType.ComputerVsComputer:
                if (showBlackDifficultyChooser.value || showWhiteDifficultyChooser.value) return true;
                break;
        }

        return false;
    });

    const showDifficutlyChooser = computed(() => {
        if (gameType.value === EGameType.PlayerVsPlayer) return false;
        if (!showBlackDifficultyChooser.value && !showWhiteDifficultyChooser.value) return false;
        return true;
    });

    const showBlackDifficultyChooser = computed(() => {
        if (gameType.value === EGameType.PlayerVsPlayer) return false;
        return blackPlayer.value.difficulty === null;
    });

    const showWhiteDifficultyChooser = computed(() => {
        if (gameType.value === EGameType.PlayerVsPlayer || gameType.value === EGameType.PlayerVsComputer) return false;
        return whitePlayer.value.difficulty === null;
    });

    const opponentsStones = computed(() => {
        if (!gameState.value) return [];

        const stones = gameState.value.occupiedPoints.filter((point) => {
            // if not opponent player - skip
            if (point.player === gameState.value.player) return false;

            // if part of mill - skip
            if (isInFormedMill(point)) {
                return false;
            }

            return true;
        });

        // If no stones - return all opponent stones
        if (stones.length === 0) {
            return gameState.value.occupiedPoints.filter((point) => point.player !== gameState.value.player);
        }

        return stones;
    });

    const formedMillStones = computed(() => {
        return gameState.value.occupiedPoints.filter((point) => isInFormedMill(point));
    });

    const nearbyFreePoints = computed(() => {
        if (selectedFigure.value === null) return [];
        if (!selectedMap.value) return [];

        const points: Array<String> = [];

        if (countPieces(gameState.value.player) === 3) {
            for (const point of selectedMap.value.map_data.points) {
                if (!gameState.value.occupiedPoints.find((occupied_point) => occupied_point.point === point)) {
                    points.push(point);
                }
            }
        } else {
            for (let index = 0; index < selectedMap.value.map_data.connections.length; index++) {
                const connection = selectedMap.value.map_data.connections[index];

                if (connection[0] === selectedFigure.value.point) {
                    for (let index = 1; index < connection.length; index++) {
                        // If point is occupied by other player figure - skip
                        if (gameState.value.occupiedPoints.find((occupied_point) => occupied_point.point === connection[index])) continue;

                        const point = connection[index];
                        points.push(point);
                    }
                }
            }
        }

        return points;
    });

    const mills = computed(() => {
        const mills = new Map<String, Array<Array<String>>>();

        if (!selectedMap.value) return mills;

        for (const point of selectedMap.value.map_data.points) {
            mills.set(point, []);
        }
        for (const mill of selectedMap.value.map_data["mills"]) {
            if (mill.length === 3) {
                // A mill is formed by 3 points
                for (const point of mill) {
                    mills.get(point)?.push(mill);
                }
            }
        }

        return mills;
    });

    const calculatedPoints = computed<Array<ICalculatedPoint>>(() => {
        if (!selectedMap.value) return [];

        const points = [];

        for (let index = 0; index < selectedMap.value.map_data.points.length; index++) {
            const point = selectedMap.value.map_data.points[index];

            const coords = normalizeChessCord(separateChessCoordToXY(point)!);
            if (!coords) continue;

            points.push({
                point: point,
                x: coords.x,
                y: coords.y,
            });
        }

        return points;
    });

    const calculatedConnections = computed<Array<ICalculatedConnection>>(() => {
        if (!selectedMap.value) return [];

        const connections = [];

        for (let index = 0; index < selectedMap.value.map_data.connections.length; index++) {
            const connection = selectedMap.value.map_data.connections[index];
            if (typeof connection[0] === "number") continue;

            const coords1 = normalizeChessCord(separateChessCoordToXY(connection[0])!);
            if (!coords1) continue;

            let coords2;

            for (let index = 1; index < connection.length; index++) {
                const point = connection[index];
                if (typeof point === "number") continue;

                coords2 = normalizeChessCord(separateChessCoordToXY(point)!);
                if (!coords2) continue;

                connections.push({
                    x1: coords1.x,
                    y1: coords1.y,
                    x2: coords2.x,
                    y2: coords2.y,
                });
            }
        }

        return connections;
    });

    const calculatedFigures = computed<Array<ICalculatedFigure>>(() => {
        if (!gameState.value) return [];

        const figures = [];

        for (let index = 0; index < gameState.value.occupiedPoints.length; index++) {
            const point = gameState.value.occupiedPoints[index];

            const coords = normalizeChessCord(separateChessCoordToXY(point.point)!);
            if (!coords) continue;

            figures.push({
                x: coords.x,
                y: coords.y,
                color: point.player,
                point: point,
            });
        }

        return figures;
    });

    function handlePointClick(point: String) {
        if (canRemoveOpponentsPiece.value) return;

        // If can place figure - place it
        if (gameType.value !== EGameType.ComputerVsComputer && gameState.value.unplacedPieces[gameState.value.player] > 0) {
            gameState.value.unplacedPieces[gameState.value.player]--;
            placeFigure(point);
        } else {
            if (!selectedFigure.value) return;
            if (!nearbyFreePoints.value.includes(point)) return;
            selectedFigure.value.point = point;
        }

        selectedFigure.value = null;

        if (isInFormedMill({ point: point, player: gameState.value.player })) {
            canRemoveOpponentsPiece.value = true;
        } else {
            canRemoveOpponentsPiece.value = false;
            setOpponentPlayer();
        }
    }

    function setOpponentPlayer() {
        if (gameState.value.player === "white") {
            gameState.value.player = "black";
        } else {
            gameState.value.player = "white";
        }
    }

    function placeFigure(point: String) {
        gameState.value.occupiedPoints.push({
            point: point,
            player: gameState.value.player,
        });
    }

    function isInFormedMill(point: ITakenPoint) {
        if (!mills.value) return false;

        const formedMills = mills.value.get(point.point);
        if (!formedMills) return false;

        for (const mill of formedMills) {
            let count = 0;
            for (const millPoint of mill) {
                if (gameState.value.occupiedPoints.find((occupied_point) => occupied_point.point === millPoint && occupied_point.player === point.player)) {
                    count++;
                }
            }
            if (count === 3) {
                return true;
            }
        }
        return false;
    }

    function countPieces(player: TPlayer) {
        return gameState.value.occupiedPoints.filter((point) => point.player === player).length + gameState.value.unplacedPieces[player];
    }

    function selectFigure(figure: ITakenPoint | null) {
        if (loading.value) return;

        if (!figure) {
            selectedFigure.value = null;
            return;
        }

        if (canRemoveOpponentsPiece.value && opponentsStones.value.includes(figure)) {
            canRemoveOpponentsPiece.value = false;

            gameState.value.occupiedPoints = gameState.value.occupiedPoints.filter((occupied_point) => {
                return occupied_point.point !== figure.point;
            });

            setOpponentPlayer();
        } else {
            if (gameState.value.unplacedPieces[gameState.value.player] > 0) return;
            if (figure.player !== gameState.value.player) return;
            selectedFigure.value = figure;
        }
    }

    function calculateGap(width?: number, height?: number) {
        if (!micaSize.value || !micaRef.value) return 0;
        const size = width ? width : getContainerSize();
        micaGap.value = size / (micaSize.value - 1);
    }

    function separateChessCoordToXY(coord: String): IPoint | null {
        const regArray = coord.match(/([a-zA-Z]+)(\d+)/);
        if (!regArray) return null;

        const [letters, numbers] = regArray.slice(1);
        const letterAsNumber = letters.toLowerCase().charCodeAt(0) - "a".charCodeAt(0);

        return {
            x: letterAsNumber,
            y: parseInt(numbers) - 1,
        };
    }

    function normalizeChessCord(coord: IPoint): IPoint {
        return {
            x: coord.x,
            y: getMicaSize() - coord.y - 1,
        };
    }

    function getMicaSize(): number {
        if (!selectedMap.value) return 0;

        let max = 0;

        for (let index = 0; index < selectedMap.value.map_data.points.length; index++) {
            const point = selectedMap.value.map_data.points[index];

            const coords = separateChessCoordToXY(point);
            if (!coords) continue;

            if (coords.x > max) max = coords.x;
            if (coords.y > max) max = coords.y;
        }

        return max + 1;
    }

    function getContainerSize() {
        if (!micaRef.value) return 0;
        return micaRef.value.clientWidth;
    }

    function handleDifficultyClick(player: TPlayer, difficulty: EDifficulty) {
        if (player === "white") {
            whitePlayer.value.difficulty = difficulty;
        } else {
            blackPlayer.value.difficulty = difficulty;

            if (gameType.value === EGameType.ComputerVsComputer) {
                makeMove();
            }
        }
    }

    function difficultyToDepth(difficulty: EDifficulty) {
        switch (difficulty) {
            case EDifficulty.Easy:
                return 3;
            case EDifficulty.Medium:
                return 5;
            case EDifficulty.Hard:
                return 7;
            default:
                return 3;
        }
    }

    async function makeMove() {
        if (!selectedMap.value) return;

        loading.value = true;

        const difficulty = gameState.value.player === "white" ? whitePlayer.value.difficulty : blackPlayer.value.difficulty;
        const depth = difficultyToDepth(gameState.value.player === "white" ? whitePlayer.value.difficulty! : blackPlayer.value.difficulty!);
        const timeout = gameState.value.player === "white" ? whitePlayer.value.timeout : blackPlayer.value.timeout;

        try {
            const data = await Client.calculateMove({
                mapName: selectedMap.value.map_name,
                timeout: timeout,
                difficulty: difficulty ?? EDifficulty.Easy,
                depth: depth,
                gameState: gameState.value,
            });

            gameState.value = data.gameState;
            lastEval.value = data.eval;
            madeMove.value = data.move;
            console.log("move made", data.move);
        } catch (error) {
            if (axios.isCancel(error)) {
                console.log("Request canceled: ", error.message);
            } else {
                console.error("Calculate move error: ", error);
            }
        }

        loading.value = false;
    }

    function setSelectedMap(map: IMapObject) {
        selectedMap.value = map;

        micaSize.value = getMicaSize();
        calculateGap();
    }

    function handleGameCardClick(game: EGameType) {
        gameType.value = game;
    }

    function resetGame() {
        Client.cancelRequest("Game reset");

        gameState.value = {
            player: "white",
            unplacedPieces: {
                black: 9,
                white: 9,
            },
            occupiedPoints: [],
        };

        canRemoveOpponentsPiece.value = false;
        selectedFigure.value = null;
        gameType.value = null;
        loading.value = false;
        whitePlayer.value.difficulty = null;
        blackPlayer.value.difficulty = null;
    }

    function setupKeyboardShortcuts() {
        window.addEventListener("keydown", (event) => {
            if (event.key === "Escape") {
                event.preventDefault();
                resetGame();
            }
        });
    }

    watch(
        gameState,
        async (newValue: IGameState, oldValue: IGameState) => {
            if (!newValue) return;

            const blackStones = countPieces("black");
            const whiteStones = countPieces("white");

            console.log("blacks", blackStones);

            // REFACTOR - Make it pretty
            if (blackStones === 2) {
                alert("White player won!");
            } else if (whiteStones === 2) {
                alert("Black player won!");
            }

            // Reset game if 2 stones left
            if (blackStones === 2 || whiteStones === 2) {
                resetGame();
                return;
            }

            if (gameType.value === EGameType.ComputerVsComputer) {
                makeMove();
            } else if (gameType.value === EGameType.PlayerVsComputer) {
                if (newValue.player === "black") {
                    makeMove();
                }
            }
        },
        { deep: true }
    );

    onMounted(async () => {
        mapsData.value = await Client.maps();

        if (mapsData.value && mapsData.value.length > 0) {
            setSelectedMap(mapsData.value[0]);
        }

        setupKeyboardShortcuts();
    });
</script>

<style scoped lang="scss">
    @mixin flex($direction: row, $justify: center, $align: center) {
        display: flex;
        flex-direction: $direction;
        justify-content: $justify;
        align-items: $align;
    }

    @mixin full-screen {
        width: 100vw;
        height: 100vh;
    }

    @mixin full {
        width: 100%;
        height: 100%;
    }

    @mixin shadow {
        box-shadow: 0px 0px 10px 0px rgba(255, 255, 255, 0.2);
    }

    @mixin padding($padding: 40px) {
        padding: $padding;
    }

    @mixin backdrop-filter {
        backdrop-filter: blur(10px);
    }

    @mixin border-radius($radius: 10px) {
        border-radius: $radius;
    }

    @mixin border-rounded {
        border-radius: 50%;
    }

    @mixin transition($transition: all) {
        transition: $transition 100ms ease-in-out;
    }

    @mixin translateCenter {
        position: absolute;
        transform: translate(-50%, -50%);
    }

    #home {
        @include flex();
        @include full();
        @include padding();

        gap: 40px;
        background-color: rgb(24, 87, 104);

        .honk {
            text-shadow: 0 0 5px rgba(255, 255, 255, 0.5);
        }

        .nig {
            text-shadow: 0 0 5px rgba(0, 0, 0, 0.9);
        }

        .unclickable {
            pointer-events: none;
        }

        .info {
            @include flex(column, flex-start);
            @include border-radius();
            @include padding();
            @include shadow();

            height: 100%;
            gap: 40px;
            background-color: rgb(82, 167, 48);
            width: 350px;

            .tex {
                color: white;
                font-weight: bold;
                font-size: 25px;
            }

            .title {
                font-size: 45px;
                font-weight: bold;
                color: white;
                margin-top: 0;
            }

            .game-type-chooser,
            .difficulty-chooser {
                @include flex(column, center, start);

                button {
                    @include backdrop-filter();
                    @include border-radius();
                    @include flex();
                    @include transition();

                    width: 200px;
                    height: 60px;
                    font-size: 18px;
                    font-weight: bold;
                    margin-bottom: 20px;
                    color: white;
                    border-color: white;

                    background-color: rgba(158, 3, 3, 0);

                    cursor: pointer;

                    &:hover {
                        @include shadow();
                        transform: scale(1.05);
                    }

                    &:active {
                        transform: scale(0.95);
                    }
                }
            }

            .unplaced-stones {
                @include flex(column);
                gap: 20px;
                .black,
                .white {
                    width: auto; // Adjusted this line to make width auto

                    p {
                        font-size: 24px;
                        font-weight: bold;
                        margin: 0; // Removed padding and margin to make it more compact
                    }
                }
                .figure {
                    @include flex();
                }
            }

            .lastEval {
                @include flex(column);

                p {
                    font-size: 17px;
                    font-weight: bold;
                    padding: 0;
                    margin: 0;
                    color: white;
                }
            }
        }

        .mica-container {
            @include border-radius();
            @include padding();

            background-color: rgb(225, 225, 225);

            .mica {
                @include full();

                position: relative;
                .point {
                    @include border-radius(50%);
                    @include transition(background-color);
                    @include flex();
                    @include translateCenter();

                    width: 48px;
                    height: 48px;

                    z-index: 10;

                    cursor: pointer;

                    .point-inner {
                        @include border-rounded();
                        @include transition();

                        width: 16px;
                        height: 16px;
                        background-color: rgb(82, 167, 48);
                    }

                    &.nearby {
                        .point-inner {
                            transform: scale(1.2);
                            background-color: rgb(169, 48, 169);
                        }

                        &:hover {
                            .point-inner {
                                background-color: rgb(196, 64, 196);
                            }
                        }

                        &:active {
                            .point-inner {
                                background-color: rgb(144, 32, 144);
                            }
                        }
                    }

                    &:hover {
                        .point-inner {
                            background-color: rgb(96, 96, 96);
                        }
                    }

                    &:active {
                        .point-inner {
                            background-color: rgb(48, 48, 48);
                        }
                    }
                }

                .connection {
                    position: absolute;
                    border: 2px solid gray;

                    translate: -2px -2px;
                }
            }
        }

        .figure {
            @include shadow();
            @include backdrop-filter();
            @include transition();
            @include border-rounded();

            width: 48px;
            height: 48px;

            z-index: 20;

            cursor: pointer;

            &.absolutely-positioned {
                position: absolute;
                position: absolute;
                transform: translate(-50%, -50%);
            }

            &.selected {
                transform: translate(-50%, -50%) scale(1.2);
            }

            &.black {
                box-shadow: 0 0 10px 5px rgba(0, 0, 0, 0.35);
                background-color: rgb(0, 0, 0);

                &.isInFormedMill {
                    background-color: rgba(2, 13, 2, 0.774);
                }

                &.canRemove {
                    background-color: rgba(187, 1, 1, 0.5);
                }
            }

            &.white {
                box-shadow: 0 0 10px 5px rgba(255, 255, 255, 0.35);
                background-color: rgb(255, 255, 255);

                &.isInFormedMill {
                    background-color: rgba(5, 181, 5, 0.183);
                }

                &.canRemove {
                    background-color: rgba(247, 43, 43, 0.5);
                }
            }

            &:hover {
                transform: brightness(1.2);
            }

            &:active {
                transform: brightness(0.8);
            }
        }
    }

    .full-screen-overlay {
        @include flex();
        @include full-screen();

        position: fixed;
        top: 0;
        left: 0;
        z-index: 9999;

        background-color: rgba(0, 0, 0, 0.25);
    }

    // Responsive CSS
    @media (max-aspect-ratio: 1/1) {
        #home {
            flex-direction: column;
            padding: 20px;

            .info {
                width: 100%;
                height: auto;

                padding: 20px;

                flex-direction: row;

                .unplaced-stones {
                    flex-direction: row;
                    gap: 20px;
                }

                .lastEval {
                    flex-direction: row;
                }
            }
        }

        .game-type-cards,
        .difficulty-cards {
            flex-direction: column !important;
        }

        .mica-container {
            width: 100%;
            aspect-ratio: 1;
        }
    }

    @media (min-aspect-ratio: 1/1) {
        .mica-container {
            height: 100%;
            aspect-ratio: 1;
        }
    }

    @media (max-width: 540px) {
        .figure {
            width: 32px !important;
            height: 32px !important;
        }

        .point {
            width: 32px !important;
            height: 32px !important;

            .point-inner {
                width: 12px !important;
                height: 12px !important;
            }
        }
    }
</style>
