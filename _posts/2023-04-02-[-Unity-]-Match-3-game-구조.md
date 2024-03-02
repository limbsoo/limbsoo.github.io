---
categories:
  - Unity
tags:
  - unity
  - gameEngine
  - match-3Game
  - geneticAlgorithm
  - cSharp
---
# 분석
___


Match-3 게임은 퍼즐 게임의 한 유형으로, 사용자가 여러 블록으로 구성된 맵에서 부여된 목표를 제한조건 내에 달성해야 하는 게임이다. 사용자는 블록 이동, 스왑(swap)을 통해 3개 이상의 동일 종류의 블록이 수직 혹은 수평 방향으로 연속 배치된 형태, 매치(match)를 이루고 매치를 이룬 블록들이 파괴되는 특성을 활용, 게임을 진행한다. 

Match-3 게임은 레벨별 하나의 맵과 제한조건, 목표로 구성되어 있으며, 단일 게임에서 1,000여 개 이상의 레벨이 존재할 수 있다. 따라서 모든 맵을 사람이 구성하는 것에 어려움이 있어, 맵 생성을 자동으로 생성하기 위한 알고리즘을 구상한다.



Match-3 게임은 아래와 같은 구조를 가지고 있으며,
<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/4fc1377c-9417-4cd5-a0c9-076316b6f09f" alt width=700>
<em></em>
</center>


맵을 구성하는 블록은 아래 그림과 같이 크게 매치 가능한 일반 블록과 매치 불가능한 특수 블록으로 나뉜다.



<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/dade7ff9-0300-41dc-a0c2-fe85375d2c05" alt width=600>
<em></em>
</center>



Match-3 게임 맵은 그리드로, 각각의 셀로 구성되어 있다.

```c#
public class MatchGrid
{
    private GameObjectsSet goSet;
	public LevelConstructSet LcSet { get; private set; }
    private int  vertSize;
    private int horSize;
	public List<GridCell> Cells { get; private set; }
}
```

```c#
public class GridCell : TouchPadMessageTarget, IDisposable
{
    private Sprite Left;
    private Sprite Right;
    private Sprite Top;
    private Sprite Bottom;

    public int Row { get; private set; }
    public int Column { get; private set; }


    public MatchObject Match { get { return   GetComponentInChildren<MatchObject>(); } }
    public FallingObject Falling { get { return GetComponentInChildren<FallingObject>(); } }
    public DynamicBlockerObject DynamicBlocker { get { return GetComponentInChildren<DynamicBlockerObject>(); } }
    public OverlayObject Overlay { get { return GetComponentInChildren<OverlayObject>(); } }
    public BlockedObject Blocked { get { return GetComponentInChildren<BlockedObject>(); } }
	
    public List<GridCell> fillPathToSpawner;
    public Spawner spawner;
    public NeighBors Neighbors { get; private set; }
}
```


따라서 게임이 시작되면 맵을 생성, 그리드의 각 셀에 블록들을 배치

```c#

MainGrid = create(MainLCSet, GridContainer);

MatchGrid create(LevelConstructSet lC, Transform cont)
{
    MatchGrid g = new MatchGrid(lC, GOSet, cont, SortingOrder.Base, GMode);

	g.FillGrid(true);
	// create spawners
	g.haveFillPath = lC.HaveFillPath(g);
	if (g.haveFillPath)
	{
		foreach (var item in lC.spawnCells)
		{
			if (g[item.Row, item.Column]) g[item.Row, item.Column].CreateSpawner(spawnerPrefab, Vector2.zero);
		}

	}
	else
	{
		g.Columns.ForEach((c) =>
		{
			c.CreateTopSpawner(spawnerPrefab, spawnerStyle, GridContainer.lossyScale, transform);
		});
	}

	// create pathes to spawners
	CreateFillPath(g);

	g.Cells.ForEach((c) =>
	{
		c.CreateBorder();
	});

    return g;
}


```


맵 생성 후, 현재 상태, MbState를 통해 현재 상황을 판단하여 게임의 진행 여부를 결정한다.

```c#
//현재 상태를 판단하는 함수
private void Update()
{
    if (!canPlay) return;
    if (WinContr.Result == GameResult.Win) return;
    if (WinContr.Result == GameResult.Loose) return;

    WinContr.UpdateTimer(Time.time);

    // check board state
    switch (MbState)
    {
        case MatchBoardState.ShowEstimate:
            ShowEstimateState();
            break;

        case MatchBoardState.Fill:
            FillState();
            break;

        case MatchBoardState.Collect:
            CollectState();
            break;
    }
}

```

- ShowEstimateState() : 게임의 진행 상황을 판단하여 충족된 조건(성공, 실패, 유지)이 존재하는 지 확인
	- 성공 : 게임 종료 후, 성공 화면 출력
	- 실패 : 게임 종료 후, 실패 화면 출력
	- 유지
		- 매치 가능한 블록 존재 X : 블록을 랜덤하게 섞음
		- 매치 가능한 블록 존재 O : 입력 대기

- FillState()는 그리드 내에 빈 셀 존재 시, 해당 셀을 채운다.

- CollectState()는 매치된 블록이 존재한다면 해당 블록을 파괴하고 목표에 해당하는 블록이 파괴된 경우 점수를 올린다.








