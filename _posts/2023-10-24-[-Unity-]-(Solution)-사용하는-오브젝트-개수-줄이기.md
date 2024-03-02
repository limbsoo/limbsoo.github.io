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
# 문제
___

Match-3 게임에서 유전 알고리즘을 통한 자동 맵 생성 구현을 위해 유전자로 맵의 크기와 같은 배열을 사용, 배열 값에 따라 각 맵에 위치한 블록의 종류를 정한다. 이를 통해 맵 생성 시 초기 블록이 배치되고, 게임이 진행됨에 따라 파괴, 생성 과정이 진행된다.

```c#

internal GridObject SetObject(GridObject prefab)
{
    if (!prefab) return null;

    int a = prefab.ID;
    GridObject gO = prefab.Create(this, MBoard.TargetCollectEventHandler);
    if (gO && !GameObjectsSet.IsDisabledObject(prefab.ID)) sRenderer.enabled = true;
    return gO;
}

internal void DestroyObjects()
{
    gridObjects = null;

    gridObjects = GetComponentsInChildren<GridObject>();
    foreach (var item in gridObjects)
    {
        DestroyImmediate(item.gameObject);
    }
}
```


그러나 유전 알고리즘을 진행 시, 원하는 평균 스왑 횟수를 가진 특수 블록 배치 맵을 생성 과정에서 한 세대에 10분 이상의 시간이 소요되는 현상이 발생하였다.

디버깅을 통해 소요 시간을 확인했을 때, 오브젝트 생성, 파괴에서 다른 곳에 비해 많은 시간을 소모함을 알 수 있었고 이를 해결하고자 했다.


# 해결
___




처음 오브젝트 생성, 파괴에서 시간이 많이 소요되는 것을 확인했을 때, 유니티의 가비지 컬렉터(GC) 문제로 판단해 이를 고치고자 하였다. 

가비지 컬렉터는 메모리를 자동으로 관리하여 필요 없는 클래스 인스턴스를 메모리에서 즉시 삭제하지 않고, 일정 조건을 만족한 후에 삭제하므로, 이러한 연유로 인해 오브젝트 생성 및 파괴에 시간이 많이 소요되는 것으로 판단하여 임의로 dispose하는 코드를 통해 인스턴스를 제거하려 했으나, 비슷한 결과를 보였다.

```c#
public sealed class DisposableScope : IDisposable
{
    // you can use ConcurrentQueue if you need thread-safe solution
    private readonly Queue<IDisposable> _disposables = new();

    public T Using<T>(T disposable) where T : IDisposable
    {
        _disposables.Enqueue(disposable);
        return disposable;
    }

    public void Dispose()
    {
        foreach (var item in _disposables)
            item.Dispose();
    }
}
```

기존 유니티 메모리 프로파일러에서 프로그램 시작 시, 멈추는 결과와 가비지 컬렉터가 작동한지 않는 것으로 판단하였으나, 다시 확인해본 결과 가비지 컬렉터는 작동을 계속하고 있었다. 따라서 문제는 오브젝트 생성과 파괴가 너무 빈번하게 일어나서 발생하는 것으로 판단되며, 이를 해결하기 위해 오브젝트 풀링을 사용하였다.

오브젝트의 pool(웅덩이)을 생성, 필요할 때마다 객체를 꺼내 사용하는 알고리즘이다. 따라서, 미리 사용할 블록 수만큼만 미리 생성하고, 사용할 때만 꺼낸 후, 사용이 끝나면 다시 집어넣는 형태를 구현하고자 했다.

```c#

public Queue<List<GridCell>> llg = new Queue<List<GridCell>> ();

for(int i = 0; i < n; i++) 
{
    llg.Enqueue(new List<GridCell> ());
}
```


그러나 문제는 임의로 이러한 블록 수를 정했을 때, 맵의 크기에 맞게 정하는 경우 일정 반복 시 할당 해제 전에 빈 큐를 호출하여 오류가 발생하였고, 이를 해결하려면 더 많은 블록 수를 사용해야 하며 적절한 블록 수를 예측하기 어렵다.

그래서 생각한 것은 그리드의 각 셀에 오브젝트를 블록 종류 별로 위치시킨 후, 비활성화하여 해당 맵에 블록이 배치될 때, 해당 블록을 활성화하여 사용하도록 했다. 따라서 맵의 크기에 맞게 적절한 오브젝트 개수를 생성할 수 있었다.

```c#

List<blockObject> Objects = new List<blockObject>();

// 종류별 block object 생성
Objects = makeBlocks(GameObjectsSet);

//셀 별 오브젝트 풀 생성
for(int i = 0;i<Cells.Count;i++) 
{
	Cells[i].objectPools = new List<blockObject>();
	Cells[i].setObjectPool(Objects);
}

...
// 사용 시 
Cells[i].objectPools[사용할 블록 ID]gameObject.SetActive(true);

```


이러한 알고리즘을 구현, 기존 한 세대에 10분 이상 걸리는 현상을 약 2분으로 감소하였다.






