---
published: "true"
title: "[데이터구조] 에너지를 수거하며 최단 경로 탐색"
date: 2025-07-19 21:10:00 +0900
author: oi-RYH
categories: 
    - DataStruct
toc: true
toc_sticky: true
---
# 에너지를 수거하면서 최단 경로를 찾아라?
처음 이 과제를 받았을 때는 단순하게 그리디 알고리즘으로 가장 가까운 에너지 탐색 -> 그 다음 에너지 -> 그 다음 에너지 이런 방식으로 가면 될 줄 알았다.
하지만 좀 더 생각해보니까, 최소 거리를 계속 탐색해도 궁극적인 최단 거리가 되진 않는걸 알았다. [ E . . S E . D ] 같은 경우...
그래서 어떤 방식으로 탐색을 해야하나 고민을 해봤는데 결국 시작점, 도착점, 모든 에너지를 정점으로 두고 정점간의 거리를 모두 계산 한 뒤에 그 중에서 최단 거리를 구해야 한다는 것을 꺠달았다. 이 방식으로 짜기 위해서 시작점이 0번, 도착점이 무조건 마지막으로 도착되고 그 사이에 에너지들의 정점을 가지며 각 정점들의 가중치가 지도 상의 거리로 구성된 그래프를 하나 더 그리기로 했다.

아래에는 내 전체 코드가 나와있다.

### whole code
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#define SIZE 81
#define MAX_VERTICES SIZE * SIZE
#define INF 999999999

// map: 미로
// mapSize: [0]에는 사용할 크기의 row, [1]에는 col 값을 저장
// MINIMUM: 최종 거리를 저장
int map[SIZE][SIZE] = { 0 };
int mapSize[2] = { 0 };
int MINIMUM = 0;

// positions: 모든 에너지, 출발/목적지의 좌표를 갖고 있는 배열
// 모든 경우에 대해 dijkstra를 해봐야하기 떄문에 따로 좌표를 저장하고 있음
int positions[MAX_VERTICES] = { 0 };
int BESTENERGY[MAX_VERTICES];
int energyCnt = 0;

// prev: 모든 경우에 대한 경로를 저장해놓는 배열
// weight: [from][to]에 대한 거리를 저장하는 2차원 배열로 표현된 그래프
// dist: dijkstra 실행 후 결정된 길이를 바로 가져오기 위해 전역변수 선언
int prev[MAX_VERTICES * MAX_VERTICES][MAX_VERTICES] = { 0 };
int weight[MAX_VERTICES][MAX_VERTICES] = { 0 };
int dist[MAX_VERTICES] = { 0 };

// dijkstra functions
void setObstacle(int _row, int _col);
void setEnergy(int _row, int _col);
int dijkstra(int _start, int _goal, int _number);
int isConnected(int _now, int _next);
int isObstacle(int _coord);

// TSP functions
void minPath();
int nextPermutation(int* _energyArr, int* _energyCnt);

// JUST PRINT MAZE
void drawPath();
void printMaze();
void AdvancedPrintMaze();

int main ()
{
    int row, col, set;
    int start[2];
    int goal[2];

    // 시작과 도착의 좌표를 먼저 입력 받음
    printf("Enter start point (row, col): ");
    scanf("%d %d", &start[0], &start[1]);

    printf("Enter destination point (row, col): ");
    scanf("%d %d", &goal[0], &goal[1]);

    // 파일 입력 시작
    FILE* f = fopen("map.txt", "rt");

    // 사용할 맵의 크기 지정
    fscanf(f, "%d %d", &mapSize[0], &mapSize[1]);

    // 장애물, 에너지 입력
    while (fscanf(f, "%d %d %d", &row, &col, &set) == 3) 
    {
        if (set == 1) setObstacle(row, col);
        else if (set == 2) setEnergy(row, col);
    }

    // 출발지와 목적지 정점 생성
    positions[0] = start[0] * mapSize[1] + start[1];
    positions[energyCnt + 1] = goal[0] * mapSize[1] + goal[1];

    for (int i = 0; i < energyCnt + 2; i++)
    {
        for (int j = 0; j < energyCnt + 2; j++)
        {
            if (i != j)
            {
                // 경로가 없을 시에는 dijkstra에서 0을 반환
                // 경로가 있다면 거리(가중치)를 저장하고, 없다면 "경로 없음" 출력 후 파일 닫고 종료
                int possible = dijkstra(positions[i], positions[j], i * (energyCnt + 2) + j);
                if (possible == 1) weight[i][j] = dist[positions[j]];
                else 
                {
                    printf("CANT");
                    fclose(f);
                    return 0;
                }
            }
        }
    }

    // 현재까지 추적한 모든 경로 중 최소 탐색
    minPath();

    // 선정된 경로들을 맵 행렬에 역추적
    drawPath();

    // 경로 따라서 맵 출력
    printMaze();
    // AdvancedPrintMaze();

    fclose(f);
    return 0;
}

/*-------- dijkstra functions --------*/
void setObstacle(int _row, int _col)
{
    map[_row][_col] = 1;
}

void setEnergy(int _row, int _col)
{
    // 꼭 지나가야하는 거점으로 삼아야하기 때문에(후에 그래프를 재구성하기 위해) positions에 전부 저장하고 개수 파악을 위해 energyCnt로 숫자 파악
    map[_row][_col] = 2;

    // 좌표 인코딩
    positions[energyCnt + 1] = _row * mapSize[1] + _col;
    energyCnt++;
}

int dijkstra(int _start, int _goal, int _number)
{
    int visited[MAX_VERTICES] = { 0 };

    // 사용 부분 이상을 가면 안 되기 때문에 최대 인덱스 설정
    int maxIndex = mapSize[0] * mapSize[1] - 1;

    // 모든 vertices에 대해 거리를 INF로 초기화 해둠
    for (int i = 0; i < SIZE * SIZE; i++) { dist[i] = INF; }

    // coordinate is row * size + col
    int now = _start;
    int dest = _goal;

    // prev -> 시작 지점인걸 알리기 위해 -1
    // dist -> 시작 지점은 당연히 거리가 0
    prev[_number][now] = -1;
    dist[now] = 0;

    // now와 dest의 값이 같다면 도착한것임!!
    while (now != dest)
    {
        int min = INF;

        // 거리 배열과 방문 배열을 탐색하면서 가장 짧은 거리의 노드로 이동 (거리를 확정하기 위해)
        for (int i = 0; i < mapSize[0] * mapSize[1]; i++)
        {
            if (!visited[i] && dist[i] < min)
            {
                min = dist[i];
                now = i;
            }
        }

        // min 값이 INF로 남아있다?? -> 모든 곳을 visited 했거나, 모든 경로가 확정 되었다!!
        // 모든 vertices를 방문했다면 목적지를 찾았을텐데 이 조건문에 걸렸다는 것은 dist 중에 INF가 남아있다는 것! => 해당 vertex로 접근이 불가능하다는 뜻
        // 이 map은 경로 찾기가 불가능...
        if (min == INF) return 0;

        // 경로 확정 -> 너는 방문 되었다
        visited[now] = 1;

        // where is next?
        for (int next = 0; next <= maxIndex; next++)
        {
            // next는 방문되지 않은 곳이면서 현재 vertex와 연결 되어있어야함! (장애물 판별)
            if (!visited[next] && isConnected(now, next))
            {
                // next에 저장되어있는 거리보다, 지금 있는 곳에서 next까지 가게될 거리가 더 가깝다면?
                if (1 + dist[now] < dist[next])
                {
                    // 거리를 수정해주고, next 노드는 now 노드에서 왔음을 알림!
                    dist[next] = 1 + dist[now];
                    prev[_number][next] = now;
                }
            }
        }
    }

    // 반복문이 종료 되었다 -> 목적지까지의 경로를 찾게 되었다: possible = 1;
    return 1;
}

int isConnected(int _now, int _next)
{
    int connection = 0;

    // 좌우로 인접했는지 혹은 장애물 블럭인지
    if ((_now - _next == 1 || _now - _next == -1) && (_now / mapSize[1] == _next / mapSize[1])
    && !isObstacle(_next)) connection = 1;
    // 상하로 인접했는지 혹은 장애물 블럭인지
    else if ((_now - _next == mapSize[0] || _now - _next == - mapSize[0]) 
    && !isObstacle(_next)) connection = 1;

    // 연결 여부 파악
    return connection;
}

int isObstacle(int _coord)
{
    int row = _coord / mapSize[1];
    int col = _coord % mapSize[1];

    // 이 인덱스에 장애물이 있나요?
    if (map[row][col] == 1) return 1;
    else return 0;
}


/*-------- TSP functions --------*/
void minPath()
{
    // 순수 에너지 인덱스만 담기 위한 배열
    int* energyOrder = (int*)malloc(sizeof(int) * energyCnt);

    // positions에서 출발지와 도착지를 제외하고 순열을 만들기 위함
    // 왜 필요한가? 출발지와 목적지의 순서는 바뀌면 안 됨. 중간에 거쳐가는 에너지들의 순서만 알아내면 되기 떄문
    for (int i = 0; i < energyCnt; i++) { energyOrder[i] = i + 1; }

    int min = INF;


    // do while의 사용 이유: 그냥 while을 사용하니 1 2 3 4 ... n 순으로 되어있는 에너지 순서를 따로 탐색을 한 뒤에 반복문을 돌아야함. 구현도 귀찮고 가독성도 오히려 좋지 않은 것 같다고 판단
    do
    {
        int total = 0;
        int from = 0;

        // 배열에 저장된 첫 번째 에너지부터 마지막 에너지까지 순서대로 가며 총 거리 파악
        for (int i = 0; i < energyCnt; i++)
        {
            int to = energyOrder[i];
            total += weight[from][to];
            from = to;
        }
        
        // 목적지까지의 거리도 합산
        total += weight[from][energyCnt + 1];

        // 최소 거리인지 판별
        if (total < min)
        {
            // memcpy: 최소 경로라고 파악된 에너지 순서를 배열에 복사 해놓는 역할
            MINIMUM = min = total;
            memcpy(BESTENERGY, energyOrder, sizeof(int) * energyCnt);
        }
    }
    while (nextPermutation(energyOrder, energyOrder + energyCnt));
    // nextPermuation의 두 번째 파라미터는 int 값인데 아규먼트로 주소 + 그냥 값을 넘기는 이유:
    // 두 번째 파라미터는 배열의 마지막 + 1번째 주소를 받는 역할. 그래서 energyOrder의 첫번째 + energyCnt의 값을 넘기는 것
    // 반환이 1이면 앞으로 더 탐색할 순열이 남았다는 것이고, 0이면 모든 순열을 탐색했다는 신호

    // 동적할당 취소
    free(energyOrder);
}

int nextPermutation(int* _first, int* _last)
{
    int endOfArr = _last - _first;

    // 배열의 뒷 부분부터 탐색을 기준으로, n번째보다 n - 1번째 숫자가 더 작은지 판별
    // 내림차순이 깨지는 순간 파악
    int i = endOfArr - 1;
    while (i > 0 && _first[i] <= _first[i - 1]) i--;

    // i가 0 -> 배열이 내림차순으로 정렬되어있다: 모든 경우의 순열을 파악했음
    if (i == 0) return 0;

    // i - 1번째보다 큰 요소 탐색
    int j = endOfArr - 1;
    while (_first[j] <= _first[i - 1]) j--;

    // 서로 자리를 바꿈
    int tmp = _first[i - 1];
    _first[i - 1] = _first[j];
    _first[j] = tmp;

    // 전체 배열을 뒤짐어줌 -> 뒤집지 않으면 모든 경우의 순열을 탐색하지 못함
    for (int a = i, b = endOfArr - 1; a < b; a++, b-- )
    {
        int tmp = _first[a];
        _first[a] = _first[b];
        _first[b] = tmp;
    }

    return 1;
}


/*-------- print --------*/
void drawPath()
{
    // prev를 돌면서 경로를 9로 바꿔둠!
    // 뒤에서부터 시작
    int vertexCnt = energyCnt + 2;
    int from = energyCnt + 1;

    // 목적지부터 출발지까지의 경로를 맵에 9로 표시
    for (int i = energyCnt; i >= 0; i--)
    {   
        // i가 1 이상일 때는 에너지 배열에서 에너지 좌표를 가지고오면 되지만, 0이 되는 순간은 posisions에서 시작지의 좌표를 가져와야함.
        int to = (i == 0) ? 0 : BESTENERGY[i - 1];
        int prevIndex = from * vertexCnt + to;

        // prev 배열에는 0 -> 1, 0 -> 2, ... 9 -> 7, 9 -> 8 경로가 prevIndex 하나당 전부 저장되어있음.
        // 가져올 prevIndex에 대해서만 경로를 가져오면 됨
        for (int track = positions[to]; track != positions[from]; track = prev[prevIndex][track])
        {
            // 좌표 복호화
            int row = track / mapSize[1];
            int col = track % mapSize[1];
            if (map[row][col] == 0) map[row][col] = 9;
        }

        from = to;
    }

    int to = 0;
    int prevIndex = from * vertexCnt + to;

    // 최종 목적지에도 경로 표시를 해줌
    int dRow = positions[energyCnt + 1] / mapSize[1];
    int dCol = positions[energyCnt + 1] % mapSize[1];

    if (map[dRow][dCol] == 0) map[dRow][dCol] = 9;
}

void printMaze()
{
    for (int row = 0; row < mapSize[0]; row++)
    {
        for (int col = 0; col < mapSize[1]; col++)
        {
            // 맵을 돌며 빈 공간은 ,. 장애물은 1, 에너지는 @, 경로는 +
            if (map[row][col] == 0) printf(". ");
            else if (map[row][col] == 1) printf("1 ");
            else if (map[row][col] == 2) printf("@ ");
            else if (map[row][col] == 9) printf("+ ");
        }
        puts("");
    }
    // 경로의 길이 출력
    printf("\t%d\n", MINIMUM);
}

void AdvancedPrintMaze()
{
    for (int row = -1; row < mapSize[0] + 1; row++)
    {
        for (int col = -1; col < mapSize[1] + 1; col++)
        {
            if (col == -1 || col == mapSize[1] || row == -1 || row == mapSize[1]) printf("⬛");
            else
            {
                if (map[row][col] == 0) printf("  ");
                else if (map[row][col] == 1) printf("⬜️");
                else if (map[row][col] == 2) printf("⚡️");
                else if (map[row][col] == 9) printf("👾");
            }
        }
        puts("");
    }
    printf("\t%d\n", MINIMUM);
}
```

### 코드의 구성

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#define SIZE 81
#define MAX_VERTICES SIZE * SIZE
#define INF 999999999

// map: 미로
// mapSize: [0]에는 사용할 크기의 row, [1]에는 col 값을 저장
// MINIMUM: 최종 거리를 저장
int map[SIZE][SIZE] = { 0 };
int mapSize[2] = { 0 };
int MINIMUM = 0;

// positions: 모든 에너지, 출발/목적지의 좌표를 갖고 있는 배열
// 모든 경우에 대해 dijkstra를 해봐야하기 떄문에 따로 좌표를 저장하고 있음
int positions[MAX_VERTICES] = { 0 };
int BESTENERGY[MAX_VERTICES];
int energyCnt = 0;

// prev: 모든 경우에 대한 경로를 저장해놓는 배열
// weight: [from][to]에 대한 거리를 저장하는 2차원 배열로 표현된 그래프
// dist: dijkstra 실행 후 결정된 길이를 바로 가져오기 위해 전역변수 선언
int prev[MAX_VERTICES * MAX_VERTICES][MAX_VERTICES] = { 0 };
int weight[MAX_VERTICES][MAX_VERTICES] = { 0 };
int dist[MAX_VERTICES] = { 0 };

// dijkstra functions
void setObstacle(int _row, int _col);
void setEnergy(int _row, int _col);
int dijkstra(int _start, int _goal, int _number);
int isConnected(int _now, int _next);
int isObstacle(int _coord);

// TSP functions
void minPath();
int nextPermutation(int* _energyArr, int* _energyCnt);

// JUST PRINT MAZE
void drawPath();
void printMaze();
void AdvancedPrintMaze();
```

다익스트라 알고리즘을 위한 함수 5개, 순열 해결을 위한 함수 2개, 최종 경로를 프린트하는 함수 3개가 있다! 이렇게 간단하게만 보고 메인 함수부터 살펴보자.


### main function

```c
int main ()
{
    int row, col, set;
    int start[2];
    int goal[2];

    // 시작과 도착의 좌표를 먼저 입력 받음
    printf("Enter start point (row, col): ");
    scanf("%d %d", &start[0], &start[1]);

    printf("Enter destination point (row, col): ");
    scanf("%d %d", &goal[0], &goal[1]);

    // 파일 입력 시작
    FILE* f = fopen("map.txt", "rt");

    // 사용할 맵의 크기 지정
    fscanf(f, "%d %d", &mapSize[0], &mapSize[1]);

    // 장애물, 에너지 입력
    while (fscanf(f, "%d %d %d", &row, &col, &set) == 3) 
    {
        if (set == 1) setObstacle(row, col);
        else if (set == 2) setEnergy(row, col);
    }

    // 출발지와 목적지 정점 생성
    positions[0] = start[0] * mapSize[1] + start[1];
    positions[energyCnt + 1] = goal[0] * mapSize[1] + goal[1];

    for (int i = 0; i < energyCnt + 2; i++)
    {
        for (int j = 0; j < energyCnt + 2; j++)
        {
            if (i != j)
            {
                // 경로가 없을 시에는 dijkstra에서 0을 반환
                // 경로가 있다면 거리(가중치)를 저장하고, 없다면 "경로 없음" 출력 후 파일 닫고 종료
                int possible = dijkstra(positions[i], positions[j], i * (energyCnt + 2) + j);
                if (possible == 1) weight[i][j] = dist[positions[j]];
                else 
                {
                    printf("CANT");
                    fclose(f);
                    return 0;
                }
            }
        }
    }

    // 현재까지 추적한 모든 경로 중 최소 탐색
    minPath();

    // 선정된 경로들을 맵 행렬에 역추적
    drawPath();

    // 경로 따라서 맵 출력
    printMaze();
    // AdvancedPrintMaze();

    fclose(f);
    return 0;
}
```
메인 함수는 별 특별한게 없다. 에너지, 장애물 위치가 담긴 파일을 읽어오면서 `setObstacle`, `setEnergy` 함수로 맵 배열에 표시해준다. 에너지는 따로 위치도 저장한다. 그리고 모든 정점을 저장하기 위해서 `positions` 배열에 따로 저장해준다. 다음 `for`문에서 다익스트라 함수를 통해 정점간의 거리를 구해준다. 이 때 함수에서 0을 반환하면 경로가 없다는 뜻이므로, "CANT"를 출력하고 종료한다.

그 뒤 `minPath`로 최소 경로를 구한 뒤 맵에 그리고, `drawPath`, `printMaze`를 통해 출력한다.


### dijkstra functions
```c
/*-------- dijkstra functions --------*/
void setObstacle(int _row, int _col)
{
    map[_row][_col] = 1;
}

void setEnergy(int _row, int _col)
{
    // 꼭 지나가야하는 거점으로 삼아야하기 때문에(후에 그래프를 재구성하기 위해) positions에 전부 저장하고 개수 파악을 위해 energyCnt로 숫자 파악
    map[_row][_col] = 2;

    // 좌표 인코딩
    positions[energyCnt + 1] = _row * mapSize[1] + _col;
    energyCnt++;
}

int dijkstra(int _start, int _goal, int _number)
{
    int visited[MAX_VERTICES] = { 0 };

    // 사용 부분 이상을 가면 안 되기 때문에 최대 인덱스 설정
    int maxIndex = mapSize[0] * mapSize[1] - 1;

    // 모든 vertices에 대해 거리를 INF로 초기화 해둠
    for (int i = 0; i < SIZE * SIZE; i++) { dist[i] = INF; }

    // coordinate is row * size + col
    int now = _start;
    int dest = _goal;

    // prev -> 시작 지점인걸 알리기 위해 -1
    // dist -> 시작 지점은 당연히 거리가 0
    prev[_number][now] = -1;
    dist[now] = 0;

    // now와 dest의 값이 같다면 도착한것임!!
    while (now != dest)
    {
        int min = INF;

        // 거리 배열과 방문 배열을 탐색하면서 가장 짧은 거리의 노드로 이동 (거리를 확정하기 위해)
        for (int i = 0; i < mapSize[0] * mapSize[1]; i++)
        {
            if (!visited[i] && dist[i] < min)
            {
                min = dist[i];
                now = i;
            }
        }

        // min 값이 INF로 남아있다?? -> 모든 곳을 visited 했거나, 모든 경로가 확정 되었다!!
        // 모든 vertices를 방문했다면 목적지를 찾았을텐데 이 조건문에 걸렸다는 것은 dist 중에 INF가 남아있다는 것! => 해당 vertex로 접근이 불가능하다는 뜻
        // 이 map은 경로 찾기가 불가능...
        if (min == INF) return 0;

        // 경로 확정 -> 너는 방문 되었다
        visited[now] = 1;

        // where is next?
        for (int next = 0; next <= maxIndex; next++)
        {
            // next는 방문되지 않은 곳이면서 현재 vertex와 연결 되어있어야함! (장애물 판별)
            if (!visited[next] && isConnected(now, next))
            {
                // next에 저장되어있는 거리보다, 지금 있는 곳에서 next까지 가게될 거리가 더 가깝다면?
                if (1 + dist[now] < dist[next])
                {
                    // 거리를 수정해주고, next 노드는 now 노드에서 왔음을 알림!
                    dist[next] = 1 + dist[now];
                    prev[_number][next] = now;
                }
            }
        }
    }

    // 반복문이 종료 되었다 -> 목적지까지의 경로를 찾게 되었다: possible = 1;
    return 1;
}

int isConnected(int _now, int _next)
{
    int connection = 0;

    // 좌우로 인접했는지 혹은 장애물 블럭인지
    if ((_now - _next == 1 || _now - _next == -1) && (_now / mapSize[1] == _next / mapSize[1])
    && !isObstacle(_next)) connection = 1;
    // 상하로 인접했는지 혹은 장애물 블럭인지
    else if ((_now - _next == mapSize[0] || _now - _next == - mapSize[0]) 
    && !isObstacle(_next)) connection = 1;

    // 연결 여부 파악
    return connection;
}

int isObstacle(int _coord)
{
    int row = _coord / mapSize[1];
    int col = _coord % mapSize[1];

    // 이 인덱스에 장애물이 있나요?
    if (map[row][col] == 1) return 1;
    else return 0;
}
```

정점 간의 거리를 구해주는 다익스트라 부분 함수들이다. 정점들의 위치는 전부 1차원 인덱스로 변환시켜서 저장하고 있다. 다익스트라 내부에서 이전 경로를 나타내는 `prev` 배열이 2차원으로 되어있는데, 정점들끼리 경로를 출력해야했으므로 레이어를 나눠 저장하기 위해 2차원으로 만들었다. i번 에너지와 j번 에너지가 `prev[][]`에 저장되어있고, 이 경로가 에너지+2 (시작 도착 포함)만큼 있는것이다.

### TSP functions
```c
/*-------- TSP functions --------*/
void minPath()
{
    // 순수 에너지 인덱스만 담기 위한 배열
    int* energyOrder = (int*)malloc(sizeof(int) * energyCnt);

    // positions에서 출발지와 도착지를 제외하고 순열을 만들기 위함
    // 왜 필요한가? 출발지와 목적지의 순서는 바뀌면 안 됨. 중간에 거쳐가는 에너지들의 순서만 알아내면 되기 떄문
    for (int i = 0; i < energyCnt; i++) { energyOrder[i] = i + 1; }

    int min = INF;


    // do while의 사용 이유: 그냥 while을 사용하니 1 2 3 4 ... n 순으로 되어있는 에너지 순서를 따로 탐색을 한 뒤에 반복문을 돌아야함. 구현도 귀찮고 가독성도 오히려 좋지 않은 것 같다고 판단
    do
    {
        int total = 0;
        int from = 0;

        // 배열에 저장된 첫 번째 에너지부터 마지막 에너지까지 순서대로 가며 총 거리 파악
        for (int i = 0; i < energyCnt; i++)
        {
            int to = energyOrder[i];
            total += weight[from][to];
            from = to;
        }
        
        // 목적지까지의 거리도 합산
        total += weight[from][energyCnt + 1];

        // 최소 거리인지 판별
        if (total < min)
        {
            // memcpy: 최소 경로라고 파악된 에너지 순서를 배열에 복사 해놓는 역할
            MINIMUM = min = total;
            memcpy(BESTENERGY, energyOrder, sizeof(int) * energyCnt);
        }
    }
    while (nextPermutation(energyOrder, energyOrder + energyCnt));
    // nextPermuation의 두 번째 파라미터는 int 값인데 아규먼트로 주소 + 그냥 값을 넘기는 이유:
    // 두 번째 파라미터는 배열의 마지막 + 1번째 주소를 받는 역할. 그래서 energyOrder의 첫번째 + energyCnt의 값을 넘기는 것
    // 반환이 1이면 앞으로 더 탐색할 순열이 남았다는 것이고, 0이면 모든 순열을 탐색했다는 신호

    // 동적할당 취소
    free(energyOrder);
}

int nextPermutation(int* _first, int* _last)
{
    int endOfArr = _last - _first;

    // 배열의 뒷 부분부터 탐색을 기준으로, n번째보다 n - 1번째 숫자가 더 작은지 판별
    // 내림차순이 깨지는 순간 파악
    int i = endOfArr - 1;
    while (i > 0 && _first[i] <= _first[i - 1]) i--;

    // i가 0 -> 배열이 내림차순으로 정렬되어있다: 모든 경우의 순열을 파악했음
    if (i == 0) return 0;

    // i - 1번째보다 큰 요소 탐색
    int j = endOfArr - 1;
    while (_first[j] <= _first[i - 1]) j--;

    // 서로 자리를 바꿈
    int tmp = _first[i - 1];
    _first[i - 1] = _first[j];
    _first[j] = tmp;

    // 전체 배열을 뒤짐어줌 -> 뒤집지 않으면 모든 경우의 순열을 탐색하지 못함
    for (int a = i, b = endOfArr - 1; a < b; a++, b-- )
    {
        int tmp = _first[a];
        _first[a] = _first[b];
        _first[b] = tmp;
    }

    return 1;
}
```

그 다음은 순열 탐색 함수다. 사실 이 부분은 혼자서 고민하다가 너무 어려워서 GPT의 도움을 받았다. 이런 식으로 순열의 모든 경우의 수를 탐색하는 방법을 배웠는데, 처음 보고 정말 대단하다고 생각했다. 

아무튼 이 탐색을 통해 최소 경로라고 판단된 에너지 순서는 `BESTENERGY` 배열에 붙여넣기 된다.

### print functions
```c
/*-------- print --------*/
void drawPath()
{
    // prev를 돌면서 경로를 9로 바꿔둠!
    // 뒤에서부터 시작
    int vertexCnt = energyCnt + 2;
    int from = energyCnt + 1;

    // 목적지부터 출발지까지의 경로를 맵에 9로 표시
    for (int i = energyCnt; i >= 0; i--)
    {   
        // i가 1 이상일 때는 에너지 배열에서 에너지 좌표를 가지고오면 되지만, 0이 되는 순간은 posisions에서 시작지의 좌표를 가져와야함.
        int to = (i == 0) ? 0 : BESTENERGY[i - 1];
        int prevIndex = from * vertexCnt + to;

        // prev 배열에는 0 -> 1, 0 -> 2, ... 9 -> 7, 9 -> 8 경로가 prevIndex 하나당 전부 저장되어있음.
        // 가져올 prevIndex에 대해서만 경로를 가져오면 됨
        for (int track = positions[to]; track != positions[from]; track = prev[prevIndex][track])
        {
            // 좌표 복호화
            int row = track / mapSize[1];
            int col = track % mapSize[1];
            if (map[row][col] == 0) map[row][col] = 9;
        }

        from = to;
    }

    int to = 0;
    int prevIndex = from * vertexCnt + to;

    // 최종 목적지에도 경로 표시를 해줌
    int dRow = positions[energyCnt + 1] / mapSize[1];
    int dCol = positions[energyCnt + 1] % mapSize[1];

    if (map[dRow][dCol] == 0) map[dRow][dCol] = 9;
}

void printMaze()
{
    for (int row = 0; row < mapSize[0]; row++)
    {
        for (int col = 0; col < mapSize[1]; col++)
        {
            // 맵을 돌며 빈 공간은 ,. 장애물은 1, 에너지는 @, 경로는 +
            if (map[row][col] == 0) printf(". ");
            else if (map[row][col] == 1) printf("1 ");
            else if (map[row][col] == 2) printf("@ ");
            else if (map[row][col] == 9) printf("+ ");
        }
        puts("");
    }
    // 경로의 길이 출력
    printf("\t%d\n", MINIMUM);
}

void AdvancedPrintMaze()
{
    for (int row = -1; row < mapSize[0] + 1; row++)
    {
        for (int col = -1; col < mapSize[1] + 1; col++)
        {
            if (col == -1 || col == mapSize[1] || row == -1 || row == mapSize[1]) printf("⬛");
            else
            {
                if (map[row][col] == 0) printf("  ");
                else if (map[row][col] == 1) printf("⬜️");
                else if (map[row][col] == 2) printf("⚡️");
                else if (map[row][col] == 9) printf("👾");
            }
        }
        puts("");
    }
    printf("\t%d\n", MINIMUM);
}
```

이젠 다 정해진 경로를 출력해서 보여주기만 하면 된다. 이젠 `prev`를 탐색하면서 이때까지 저장해둔 경로를 돌아가며 지도에 '9'로 저장해둔다. `drawPath`로 저장이 끝났다면 `printMaze`로 출력해주고 끝이 난다! `AdvancedPrintMaze`는 단순한 출력에서 이모지를 활용해서 거 가독성 좋게 나타내준다.

이렇게 에너지를 수거하면서 최단 경로를 찾는 과제를 해봤는데, 난이도가 살짝 있다고 생각했지만 그래도 정말 재밌게 해결했던거같다. 도중에 새로운 알고리즘도 배워보고, 만들어둔 내 코드가 제대로 작동하면서 경로를 출력하는 모습을 보니까 정말 재밌었다.