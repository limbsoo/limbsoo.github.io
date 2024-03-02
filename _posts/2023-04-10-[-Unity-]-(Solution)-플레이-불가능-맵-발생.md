---
categories:
  - Unity
tags:
  - unity
  - gameEngine
  - match-3Game
  - geneticAlgorithm
  - cSharp
  - solution
---
# 분석
___

특수 블록을 배치하여 게임 진행 시,

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/721c3c89-afd8-4a7e-808b-da771570a34e" alt width=500>
<em></em>
</center>


일부 셀 값에서 null이 발생, 게임이 진행되지 않는 경우가 발생했다.


이를 확인해본 결과, 아래 그림과 같은 형태로 특수 블록이 배치되었을 때, 빈 셀을 채우지 못해 해당 셀들이 블록이 없어 발생한 것으로 확인됬다.

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/416ae685-eb00-443e-9d72-2bbb5f727232" alt width="100%">
<em></em>
</center>




# 해결
___


따라서 이를 해결하기 위해 각 셀 간의 연결 여부를 판단한다.


그리드는 아래와 같은 구조로 각 셀에 여러 데이터들이 저장되어있으며, 주변 셀 정보도 저장되어있다.

```c#
class MatchGrid
{
	List<GridCell> Cells;
}

class GridCell
{
	int Row;
	int Column;
	
	Column<GridCell> GColumn; // List<GridCell> Cells를 Column순 정렬
	Row<GridCell> GRow; // List<GridCell> Cells를 Row순 정렬
	
	int objectID;

	List<GridCell> fillPathToSpawner;
	NeighBors Neighbors;
	bool isAvailable;
}

class Column
{
	List<GridCell> Cells;
	GridCell spawner; // col: 0의 cell
}

class NeighBors
{
	GridCell Main;
	GridCell Left;
	GridCell Right;
	GridCell Top;
	GridCell Bottom; 
	List<GridCell> Cells ;
}

```


그리고 이를 활용, 해당 셀에서 재귀적으로 탐색하여 루트를 만든다.

```c#


public void CreateFillPath(MatchGrid g)
{
    foreach (var c in g.Cells)
    {

        if (!c.spawner)
        {
			// 이동할 수 있는 블록인지 판단
            if (!c.Blocked && !c.IsDisabled && !c.MovementBlocked)
            {
                Stack<GridCell> path = new Stack<GridCell>();
                CreatePathes(map, c.pfCell, c.pfCell, path);
            }
        }

    }
}



public void CreatePathes(Map WorkMap, PFCell A, PFCell current, Stack<GridCell> currentPath)
{
    if (current.mather == null)
    {
        current.mather = current;

        if (current.Neighbors.Top != null)
        {
            finding(WorkMap, A, current.Neighbors.Top.pfCell, currentPath);
        }

        if (current.Neighbors.Left != null)
        {
            finding(WorkMap, A, current.Neighbors.Left.pfCell, currentPath);
        }

        if (current.Neighbors.Right != null)
        {
            finding(WorkMap, A, current.Neighbors.Right.pfCell, currentPath);
        }
    }
}




public void finding(Map WorkMap, PFCell A, PFCell current, Stack<GridCell> currentPath)
{
    if (current.mather == null)
    {
        current.mather = current;

        if (!current.gCell.Blocked && !current.gCell.IsDisabled && !current.gCell.MovementBlocked)
        {
            if (current.gCell.spawner)
            {
                if (A.gCell.fillPathToSpawner != null)
                {
                    if (current.gCell.fillPathToSpawner == null)
                    {
                        A.gCell.fillPathToSpawner = new List<GridCell>();
                        currentPath.Push(current.gCell);

                        foreach (var c in currentPath)
                        {
                            c.pfCell.mather = c.pfCell;
                            A.gCell.fillPathToSpawner.Add(c);
                        }
                        currentPath.Pop();
                    }

                    else
                    {
                        if (currentPath.Count + current.gCell.fillPathToSpawner.Count < A.gCell.fillPathToSpawner.Count)
                        {
                            A.gCell.fillPathToSpawner = new List<GridCell>();
                            currentPath.Push(current.gCell);
                            foreach (var c in currentPath)
                            {
                                c.pfCell.mather = c.pfCell;
                                A.gCell.fillPathToSpawner.Add(c);
                            }
                            currentPath.Pop();
                        }
                    }


                }

                else
                {
                    A.gCell.fillPathToSpawner = new List<GridCell>();
                    currentPath.Push(current.gCell);
                    foreach (var c in currentPath)
                    {
                        c.pfCell.mather = c.pfCell;
                        A.gCell.fillPathToSpawner.Add(c);
                    }

                    currentPath.Pop();
                }


            }

            else
            {
                if (current.gCell.fillPathToSpawner != null)
                {
                    if (current.gCell.fillPathToSpawner[current.gCell.fillPathToSpawner.Count - 1].spawner)
                    {
                        if (A.gCell.fillPathToSpawner != null)
                        {
                            if (currentPath.Count + current.gCell.fillPathToSpawner.Count + 1 < A.gCell.fillPathToSpawner.Count)
                            {
                                A.gCell.fillPathToSpawner = new List<GridCell>();
                                currentPath.Push(current.gCell);

                                foreach (var c in currentPath)
                                {
                                    c.pfCell.mather = c.pfCell;
                                    A.gCell.fillPathToSpawner.Add(c);
                                }

                                foreach (var c in current.gCell.fillPathToSpawner)
                                {
                                    c.pfCell.mather = c.pfCell;
                                    A.gCell.fillPathToSpawner.Add(c);
                                }
                                currentPath.Pop();
                            }
                        }

                        else
                        {

                            A.gCell.fillPathToSpawner = new List<GridCell>();
                            currentPath.Push(current.gCell);

                            foreach (var c in currentPath)
                            {
                                c.pfCell.mather = c.pfCell;
                                A.gCell.fillPathToSpawner.Add(c);
                            }

                            foreach (var c in current.gCell.fillPathToSpawner)
                            {
                                c.pfCell.mather = c.pfCell;
                                A.gCell.fillPathToSpawner.Add(c);
                            }

                            currentPath.Pop();
                        }
                    }
                }

            }
        }
    }
}

```


그래서 이를 통해 빈 셀이 채워지는데,


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/a28519ae-8378-470a-8cb7-ad4e5a54bc7a" alt width=600>
<em></em>
</center>



이러한 경우, 특수 블록이 스폰 위치와의 루트 생성을 막아 빈 셀을 채울 수 없게 된다.

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/10a3f823-2418-4fd4-8ec5-d7cc0eab9158" alt width=500>
<em></em>
</center>




이러한 경우를 방지하기 위해 fillPathToSpawner가 null인 경우, 이를 플레이할 수 없는 맵이라 판단하고, 이를 플레이하지 않는다.


```c#

if(grid.Cells[i].fillPathToSpawner == null)
{
	return false;
}
```






