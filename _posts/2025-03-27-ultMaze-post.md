---
published: "true"
title: "[데이터구조] 궁극의 미로 만들기"
date: 2025-03-27 10:40:00 +0900
author: oi-RYH
categories: 
    - DataStruct
toc: true
toc_sticky: true
---

얼마 전에 학교 과제로 완성했던 미로... 사실 과제에서는 벽을 1, 빈 공간을 0로만 출력하면 됐었다. 그런데 미로 크기가 30x30이 되자 경로를 직접 내 눈으로 확인하려는데 무수한 숫자들 덕분에 눈이 아팠다. 그래서 약간의 가시성 개선으로 벽은 하얀 네모 이모지, 빈 공간은 공백으로 출력을 하고 마무리를 했었다.

그런데 과제로 낸 코드가 미로를 만드는데 시간이 너무 오래 걸려서 속도를 좀 더 향상시키고 싶었고, 스타일도 좀 더 넣고 싶었다. 그래서 과제를 내고 코드를 좀 더 개선을 시켜봤다.

```c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>

#define MAX 30


typedef struct _coord {                         // 좌표 저장 구조체
    int row;
    int col;
} COORD;


int maze[MAX][MAX] = { 0 };                     // 미로 저장 배열
int size;   
float wallsPercent;
double wallCnt;
COORD wall;                                     // 가장 최근에 만든 벽의 좌표

int visitedMaze[MAX + 1][MAX + 1] = { 0 };      // 방문한 곳
int visitedIndex = 0;                           // 방문한 곳의 개수

COORD path[MAX * MAX];                          // 현재 경로의 스택
int pathIndex = -1;                             // 위 스택의 탑

COORD tmpPath[MAX * MAX];
int tmpPathIndex = 0;


int findPath();                                 // 길을 찾는 함수. 길이 있으면 1, 못찾으면 0, 없으면 -1 반환
void makeWalls();                               // 랜덤하게 벽을 생성
void addToVisited(COORD _cur);                  // 방문 스택에 push
void addToPath(COORD _cur);                     // 경로 스택에 push
int isDest(COORD _cur, COORD _dest);            // cur 좌표가 목적지와 일치하는지 판별
void breakWall();                               // 가장 최근의 벽 파괴
COORD whereCanIGo(COORD _cur);                  // 현재 좌표에서 갈 수 있는 곳 탐색
int isMap(COORD _cur);                          // 맵 안에 있는가?
int isWall(COORD _cur);                         // 좌표가 벽을 가르키는가?
int isVisited(COORD _cur);                      // 방문했던 곳인가?
void popToPath();                               // 경로 스택 pop
void init();                                    // 모든 스택 초기화
void printMaze();                               // 미로 출력
int howManyWalls();                             // 벽의 개수 카운팅
void copyPath();
void enterPath();
void printPath(COORD _cur);


int main () 
{
    double failCnt = 0;

    srand(time(NULL));

    printf("Enter the size you want: ");
    scanf("%d", &size);
    printf("Enter the percent of walls you want: ");
    scanf("%f", &wallsPercent);

    makeWalls();

    while (1)                     
    {
        if (findPath() == 1)                    // 길을 찾았을 경우
        {
            wallCnt = howManyWalls();           // 벽의 개수를 세고, 70% 이상이면 break 
            if (wallCnt > size * size * (wallsPercent / 100)) break;
            else makeWalls();                   // 벽이 70%가 아니라면 벽 생성
        }
        else if (failCnt > 50000)               // 너무 많이 실패한 경우 -> 불가능한 상황
        {
            printf("No possible path.\n");
            return 0;
        }
        else if (findPath() == 0) 
        {  
            breakWall();                        // 경로가 없다면 벽 파괴
            failCnt++;
        }
        copyPath();
        init();                                 // 새로운 경로를 찾아야하니 스택 초기화
        
    }

    enterPath();
    printMaze();                                // 완성된 미로 출력

    return 0;
}


int findPath()
{
    COORD _cur;                                 // 현재 좌표
    COORD _start = {0, 0};                      // 출발지
    COORD _dest = {size - 1, size - 1};         // 목적지

    _cur = _start;                              // 현재 좌표의 초기값

    addToVisited(_cur);                         // 출발지를 방문한 곳으로 스택에 push
    addToPath(_cur);

    while (1)
    {
        if (isDest(_cur, _dest) == 1) return 1; // 목적지에 도착했다면 -> 경로 발견
        _cur = path[pathIndex];                 // 경로 스택의 top을 현재 좌표로
        if (pathIndex == -1)
        {
            _cur.row = 0;
            _cur.col = 0;
        }
        COORD target = whereCanIGo(_cur);       // 현재 좌표에서 갈 수 있는 곳을 타켓 좌표로

        // debugMaze(_cur);

        if ((target.row != -1) && (target.col != -1)) 
        // 타겟 좌표가 (-1, -1)이 아닐 경우 -> 갈 곳이 있다는 의미
        {
            addToVisited(target);               // 방문 스택에 push
            addToPath(target);
            _cur = target;                      // 현재 좌표 수정
        }
        else                                    // 방문할 곳이 없다면
        {
            if (pathIndex == -1) return 0;
            popToPath();                        // 경로 후퇴
        }
    }
}


void makeWalls()
{

    while (1)
    {
        wall.row = rand() % size;
        wall.col = rand() % size;
        if (((wall.row == 0) && (wall.col == 0)) || ((wall.row == size - 1) && (wall.col == size - 1))) breakWall();
        // 출발지나 목적지에 벽이 생성된 경우엔 벽 제거
        else
        {
            if (maze[wall.row][wall.col] == 0)  // 출발지나 목적지가 아니면서 벽이 없는 곳이라면 벽 생성
            {
                maze[wall.row][wall.col] = 1;
                return;
            }
        }
    }
}


void addToVisited(COORD _cur)
{
    visitedIndex++;
    visitedMaze[_cur.row][_cur.col] = 1;        // 방문한 곳을 1로 변경
}


void addToPath(COORD _cur)
{
    pathIndex++;
    path[pathIndex].row = _cur.row;
    path[pathIndex].col = _cur.col;
}


int isDest(COORD _cur, COORD _dest)
{
    if ((_cur.row == _dest.row) && (_cur.col == _dest.col)) return 1;
    else return 0;
}


void breakWall()
{
    maze[wall.row][wall.col] = 0;
}


COORD whereCanIGo(COORD _cur)
{
    COORD trg = {0, 0};
    trg.row = _cur.row + 1;                     // 현재 좌표에서 아래로 한 칸
    trg.col = _cur.col;
    if (!isWall(trg) && isMap(trg) && !isVisited(trg)) return trg;

    trg.row = _cur.row;
    trg.col = _cur.col + 1;                     // 현재 좌표에서 오른쪽으로 한 칸
    if (!isWall(trg) && isMap(trg) && !isVisited(trg)) return trg;

    trg.row = _cur.row - 1;
    trg.col = _cur.col;                         // 현재 좌표에서 위로 한 칸
    if (!isWall(trg) && isMap(trg) && !isVisited(trg)) return trg;

    trg.row = _cur.row;
    trg.col = _cur.col - 1;                     // 현재 좌표에서 왼쪽으로 한 칸
    if (!isWall(trg) && isMap(trg) && !isVisited(trg)) return trg;
    
    trg.row = -1;                               // 갈 곳이 없을 경우
    trg.col = -1;

    return trg;
}


int isMap(COORD _cur)
{
    if (((_cur.row >= 0) && (_cur.col >= 0)) && ((_cur.row < size) && (_cur.col < size))) return 1;
    else return 0;    
}


int isWall(COORD _cur)
{
    if (maze[_cur.row][_cur.col] == 1) return 1;
    else return 0;
}

int isVisited(COORD _cur)
{
    if (visitedMaze[_cur.row][_cur.col] == 1) return 1;
    else return 0;
}


void popToPath()
{
    pathIndex--;
}


void init()
{
    for (int i = 0; i < pathIndex; i++)
    {
        path[pathIndex].row = 0;
        path[pathIndex].col = 0;
    }
    pathIndex = -1;

    for (int i = 0; i < size; i++)
    {
        for (int j = 0; j < size; j++)
        {
            visitedMaze[i][j] = 0;
        }
    }
    visitedIndex = 0;
}


void printMaze()
{
    COORD _cur;
    for (int i = -1; i <= size; i++)                // 디자인 코드
    {
        printf("👾");
    }
    puts("");

    for (int i = 0; i < size; i++)                  // 미로 출력
    {
        printf("👾");
        for (int j = 0; j < size; j++)
        {
            _cur.row = i;
            _cur.col = j;
            if (maze[i][j] >= 2) printPath(_cur); 
            else if (maze[i][j] == 0) printf("  ");
            else printf("⬜️");

        }
        printf("👾");
        puts("");
    }

    for (int i = -1; i <= size; i++)                // 디자인 코드
    {
        printf("👾");
    }
    puts("");

    float percent = 100 * wallCnt / (size * size);  // 벽이 전체의 몇 퍼센트인지 출력
    printf("walls: %.2f%%", percent);
}


int howManyWalls()
{
    wallCnt = 0;
    for (int i = 0; i < size; i++) 
    {
        for (int j = 0; j < size; j++)
        {
            if (maze[i][j] == 1) 
            {
                wallCnt++;
            }
        }
    }
    return wallCnt;
}


void copyPath()                                     // 경로를 임시 경로 스택에 저장. 미로를 출력할 떄 경로를 표시하기 위함
{
    for (int i = 0; i <= pathIndex; i++)
    {
        tmpPath[i] = path[i];
    }
    tmpPathIndex = pathIndex;
}


void enterPath()                                    // 경로는 미로에서 2로 변환
{
    for (int i = 0; i <= tmpPathIndex; i++)
    {
        maze[tmpPath[i].row][tmpPath[i].col] = i + 2;
    }
}


void printPath(COORD _cur)
{
    int direction[3][3] = { 0 };
    int i = maze[_cur.row][_cur.col] - 2;

    if ((tmpPath[i].row == 0) && (tmpPath[i].col == 0)) printf("🏃‍");
    if ((tmpPath[i].row == size - 1) && (tmpPath[i].col == size - 1)) printf("⛳️");

    if ((i < tmpPathIndex) && (i > 0))
    {
        direction[tmpPath[i - 1].row - tmpPath[i].row + 1][tmpPath[i - 1].col - tmpPath[i].col + 1] = 1;
        direction[tmpPath[i + 1].row - tmpPath[i].row + 1][tmpPath[i + 1].col - tmpPath[i].col + 1] = 1;

        if (direction[1][0] && direction[1][2])      printf("--");
        else if (direction[0][1] && direction[2][1]) printf(" │");
        else if (direction[2][1] && direction[1][2]) printf("┌-");
        else if (direction[2][1] && direction[1][0]) printf("-┐");
        else if (direction[0][1] && direction[1][0]) printf("-┘");
        else if (direction[0][1] && direction[1][2]) printf("└-");
    }
}
```

일단 기존에 있었지만 달라진 점을 설명하고 그 후에 디자인을 어떻게 했는지 설명을 하겠다.


```c
int main () 
{
    double failCnt = 0;

    srand(time(NULL));

    printf("Enter the size you want: ");
    scanf("%d", &size);
    printf("Enter the percent of walls you want: ");
    scanf("%f", &wallsPercent);

    makeWalls();

    while (1)                     
    {
        if (findPath() == 1)                    // 길을 찾았을 경우
        {
            wallCnt = howManyWalls();           // 벽의 개수를 세고, 70% 이상이면 break 
            if (wallCnt > size * size * (wallsPercent / 100)) break;
            else makeWalls();                   // 벽이 70%가 아니라면 벽 생성
        }
        else if (failCnt > 50000)               // 너무 많이 실패한 경우 -> 불가능한 상황
        {
            printf("No possible path.\n");
            return 0;
        }
        else if (findPath() == 0) 
        {  
            breakWall();                        // 경로가 없다면 벽 파괴
            failCnt++;
        }
        copyPath();
        init();                                 // 새로운 경로를 찾아야하니 스택 초기화
        
    }

    enterPath();
    printMaze();                                // 완성된 미로 출력

    return 0;
}
```
`main` 함수부터 차근차근 보자. 이제는 벽을 미로의 몇 퍼센트까지 채울지 결정할 수가 있다. 기존에는 70%로 고정되어있었지만, `wallsPercent` 변수를 통해 원하는 비율까지 벽을 생성한다.

그리고 `srand(time(NULL))`이 `main`에 자리잡게 되었다. 전에는 `makeWalls()` 내부에 있었는데 시드값 생성 위치를 옮기니 미로 생성 시간이 엄청나게 개선되었다.

`failCnt` 횟수도 줄여서 무한루프가 발생하면 보다 빠르게 코드가 종료된다.

```c
if (pathIndex == -1)
{
    _cur.row = 0;
    _cur.col = 0;
}
```
`findPath()` 함수에서는 위 부분이 추가되었다. 기존의 미로 코드에서는 `pathIndex`가 -1이 되어도 아무 문제가 발생하지 않았지만, 이번 궁극의 미로 코드에서는 치명적인 메모리 오류가 계속 발생했다. 그래서 `pathIndex == 1`이면 자동으로 `_cur` 좌표에 직접 (0, 0)을 대입시켜주면서 오류를 해결했다.


```c
void addToVisited(COORD _cur)
{
    visitedIndex++;
    visitedMaze[_cur.row][_cur.col] = 1;        // 방문한 곳을 1로 변경
}
```
다음으로는 `addToVisited()` 함수가 달라졌다. 원래 구조체 배열을 활용한 스택이였지만, 코드 수행시간 개선 도중 GPT가 방문한 곳을 그냥 미로와 같은 2차원 배열에 저장하라길래 새로 변수를 만들고 갔던 곳을 1로 저장해두었다.

```c
int isVisited(COORD _cur)
{
    if (visitedMaze[_cur.row][_cur.col] == 1) return 1;
    else return 0;
}
```
`isVisited()` 함수도 달라졌다. `visitedMaze`가 스택에서 2차원 배열로 달라졌기 때문에 기존의 `for` 반복문을 제거했다. 다음의 `init()` 함수도 달라졌지만 굳이 설명하진 않겠다.

---
지금까지는 미로를 생성하고 경로를 찾는 알고리즘에 대한 수정사항이었다. 이제부터는 미로를 출력하는 방식이 어떻게 변했는지 설명해보겠다.


```c
void printMaze()
{
    COORD _cur;
    for (int i = -1; i <= size; i++)                // 디자인 코드
    {
        printf("👾");
    }
    puts("");

    for (int i = 0; i < size; i++)                  // 미로 출력
    {
        printf("👾");
        for (int j = 0; j < size; j++)
        {
            _cur.row = i;
            _cur.col = j;
            if (maze[i][j] >= 2) printPath(_cur); 
            else if (maze[i][j] == 0) printf("  ");
            else printf("⬜️");

        }
        printf("👾");
        puts("");
    }

    for (int i = -1; i <= size; i++)                // 디자인 코드
    {
        printf("👾");
    }
    puts("");

    float percent = 100 * wallCnt / (size * size);  // 벽이 전체의 몇 퍼센트인지 출력
    printf("walls: %.2f%%", percent);
}
```
`printMaze()` 함수다. 기본적으로 미로의 테두리를 이모지 하나로 출력하도록 간소화했고, 가장 많이 변경된 부분이 경로를 출력하는 부분이다. 원래 `maze`에 2를 저장해서 경로를 나타냈지만 이제는 2씩 증가하는 등차수열을 저장해두었다. 왜 그렇게 저장했을까?

미로 출력 과정에서 내가 가장 하고싶었던건 경로를 이모티콘으로 쭉 출력하는게 아니라, 진짜 길처럼 이어진 모습을 만들어내고 싶었다. 그래서 화살표와 긴 선들 중에 어떤걸로 할지 고민했는데 화살표는 출력했을 때 너무 눈이 아플거같아서 선택하지 않았다. 그러면 이제 연속된 선들로 쭉 이어야하는데, 그럼 이제는 프린트 할 경로의 현재 좌표뿐 아니라 그 전, 후 좌표도 같이 알아야 직진인지 좌.우회전인지 알 수 있었다.

전후 좌표를 사용해 방향을 알아내기 위해, 처음에는 현재 좌표에 대한 전후 좌표의 변화량을 구하는 방식으로 방향을 출력하려했다.

그랬더니 문제가 발생했다. 현재 좌표에 대해 어떤게 전후인지 구분없이 위치만 나타내니까 회전을 할 때 어떤 방식인지 정해지지가 않는다. 예를 들어, [-1][0]에서 [0][1]로 이동했다고 해도 `row`와 `col`의 변화량이 각각 1이기 때문에 [0][-1] -> [1][0]과 구분할 수 없는 것이다.

```c
void printPath(COORD _cur)
{
    int direction[3][3] = { 0 };
    int i = maze[_cur.row][_cur.col] - 2;

    if ((tmpPath[i].row == 0) && (tmpPath[i].col == 0)) printf("🏃‍");
    if ((tmpPath[i].row == size - 1) && (tmpPath[i].col == size - 1)) printf("⛳️");

    if ((i < tmpPathIndex) && (i > 0))
    {
        direction[tmpPath[i - 1].row - tmpPath[i].row + 1][tmpPath[i - 1].col - tmpPath[i].col + 1] = 1;
        direction[tmpPath[i + 1].row - tmpPath[i].row + 1][tmpPath[i + 1].col - tmpPath[i].col + 1] = 1;

        if (direction[1][0] && direction[1][2])      printf("--");
        else if (direction[0][1] && direction[2][1]) printf(" │");
        else if (direction[2][1] && direction[1][2]) printf("┌-");
        else if (direction[2][1] && direction[1][0]) printf("-┐");
        else if (direction[0][1] && direction[1][0]) printf("-┘");
        else if (direction[0][1] && direction[1][2]) printf("└-");
    }
}
```
그럼 이제 `printPath()`함수를 보자. 기존의 변화량을 구해서 경로를 표시하는 방식에서 생긴 문제를 해결하기 위해 새로운 변수(`direction[3][3]`)를 선언했다.

`direction[3][3]` 배열은 3 x 3으로 이루어져있고 현재 좌표를 `direction[1][1]`에 대입하고 전후 좌표를 나머지 배열에 대입하면서 경로를 알아낸다. 그 과종이 아래의 `if`의 내용이다.

이렇게 방향 문제를 해결하고 출력을 해보았는데, 뭔가 경로가 이상하게 출력이 되고있는걸 발견했다.

![그렇게 출력된 경로](/assets/images/failPath.png)

이게 어떻게 잘못 나온건지 알아봤다. 계속되는 잘못된 출력을 지켜본 결과, 내가 `if` 안에서 `i`를 0부터 `tmpPathIndex`가 될 때까지 증가시키며 출력했기 때문이다. 열핏보면 이 방식이  옳은 것 같지만, 좀 더 생각해보면 우리는 지금 경로를 따라 출력하는 중이 아니라 미로 전체를 출력하면서 경로 블럭이면 함수를 호출해서 부르는 방식이기 때문에 이런 문제가 발생하고 있는 것이였다. 그래서 예를 들면, 실제로는 여러 커브길을 거치며 10번째 경로가 되었지만, i는 `printPath()`가 호출되었을 때 증가하기 때문에 실제로는 7이나 8번째 같이 더 적은, 현재 경로와 일치하지 않는 곳의 경로를 출력하는 것이다.

이 문제를 해결하기 위해 이번에는 미로에 경로 인덱스를 아예 할당하기로 했다. 

```c
void enterPath()                                    // 경로는 미로에서 2로 변환
{
    for (int i = 0; i <= tmpPathIndex; i++)
    {
        maze[tmpPath[i].row][tmpPath[i].col] = i + 2;
    }
}
```
`enterPath()`함수는 원래 경로 스택에 있던 좌표들에 2를 저장하는 역할을 했다. 그런데 방금과 같은 문제가 생긴 이후로 여기서 인덱스를 저장하기로 했다. 그런데 0이나 1부터 시작해서 쭉 할당하는건 빈 공간(0)이나 벽(1)과 구분을 못하게 되기 때문에 2부터 시작해서 `tmpPathIndex + 2`만큼 저장하기로했다.

```c
int i = maze[_cur.row][_cur.col] - 2;
```
그래서 `printPath()`를 보면 `i`를 차차 증가시키는게 아니라 현재 좌표에 저장된 값에서 아까 더한 2를 빼주면서 `tmpPath`배열의 어떤 항에 상응하는 경로를 출력해야하는지 판단하는 방식으로 바꾸었다. 그러고나니 드디어 제대로 된 경로가 출력되었다.

![드디어 잘 나온 경로](/assets/images/finalPath.png)

신이 나서 미로를 여러번 돌려보는데 이번엔 메모리 오류가 났다!

작은 값을 입력했을 때는 `bus error`가, 큰 값을 입력했을 때는 `zsh: segmentation fault`가 나타났다.

`bus error`는 `printPath`에서 `if`의 조건을 `i`가 `tmpPathIndex`보다 작은 자연수이여야 한다고 수정하니 해결되었다. 맨 처음에는 1 이상이여야하는 조건이 없었기 때문에 `i`가 -1이 되었을 때 오류가 났었다.

`zsh: segmentation fault`는 `isWall()`에서 처음 발견했다. 문제를 더 타고 올라가보니 이번에도 `pathIndex`가 -1이 되는 것이 문제였다. `findPath()`안에서 `_cur = path[pathIndex]`로 계속 현재 좌표를 수정 중이였는데 `popToPath()`를 하다가 인덱스가 -1이 될 때 랜덤하게 현재 좌표에 (645, 73)과 같은 값이 할당되는게 문제였다. 그래서 위에 설명했듯 `if`문으로 직접 초기화 해주면서 문제를 해결했다.

이렇게 과제로 했던 미로 만들기 코드를 업그레이드 해봤는데, 정말 재밌었다. 중간에 에러도 많았지만 혼자 해결해보고, 메모리 관련 문제는 GPT와 해결했다. 다음에도 뭔가 더 만들어봐야겠다.