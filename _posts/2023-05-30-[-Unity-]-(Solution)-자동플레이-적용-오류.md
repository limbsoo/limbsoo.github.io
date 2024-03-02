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


타겟 우선 알고리즘을 통한 자동 플레이를 구현 시, 이러한 문제가 적용되지 않는 문제가 발생하였다. 


# 해결
___



이러한 문제는 기존 asset의 자동 플레이 형태가 게임이 끝난 후 발생하는 것으로 이루어져 생기는 문제로, Asset의 AutoPlay는 설정된 target의 개수를 사용자가 제거했을 때 score bonus의 개념으로 남은 이동 횟수만큼 게임 세팅에 따라 랜덤하게 매치 시키거나 bomb을 생성해 score에 bonus를 주는 것으로, Asset의 AutoPlay는 target이 달성 되었을 때만 진행된다. 

현재 코드에서 상태를 수집할 때, 판단되는 상태에서 checkResult()를 통해 게임의 종료 실행 여부를 확인하는데 target이 성취되지 않았을 때, 이를 false한다.


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/d0d56c7d-1d03-4d0b-968c-b12545f3737e" alt width=700>
<em></em>
</center>




따라서 ObjectTargetsIsCollected() 함수에서 현재 target(mBoard.curTargets)들이 모두 성취되었으면(Item.Value.Achieved) return true로 autoPlay 실행된다.

이를 해결하기 위해  target 달성 코드를 주석처리 했으나 이로인해 타겟이 달성되도 확인하고 멈출 수 없는 경우가 발생하여, 종료 여부를 판단하는 함수를 추가, 타겟이 달성되면 클리어 여부를 판단하도록 했다.

```c#

public void estimateClear()
{
    foreach (var item in curTargets)
    {
        if (item.Value.Achieved) plays.isClear = true;

        else
        {
            plays.isClear = false;
            break;
        }
    }
}
```




































