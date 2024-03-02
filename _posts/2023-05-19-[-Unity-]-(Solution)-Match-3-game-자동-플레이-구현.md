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

보통 Match-3 게임은 클리어 조건 만족 시, 남은 스왑 횟수만큼 점수를 추가로 부여하는 경우가 많다. 이러한 경우 중 남은 스왑 횟수만큼 임의적으로 더 스왑시켜 추가 점수를 주는 경우가 있는데, 현재 에셋에서 이러한 방식을 통해 추가점수를 부여하므로, 이를 통해 자동 플레이를 구현하고자 한다.


게임의 상태를 판단하는 update()에서 showEstimateState()를 통해 현재 게임 상황을 판단하는데, 그 근거로 CreateGroup()을 사용한다.


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/2abe236f-1733-40b0-bdba-c9b439ce8b50" alt width=700>
<em></em>
</center>


CreateGroup()은 2가지 기능으로 이루어져, 매치 가능한 블록들이 존재하는 경우와 매치된 블록들이 존재하는 경우가 있는지 Row, column 별 탐색을 통해 판단한다. 

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/5bcc054b-ab61-4f50-8156-9d56f8e49234" alt width=700>
<em></em>
</center>

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/f508d901-9c31-485b-9d4f-171d181e3282" alt width=700>
<em></em>
</center>


스왑 횟수가 남아있고, 매치된 블록들이 없을 경우, 매치 가능한 블록들이 존재하는 지를 판단한다. 셀 간 거리 2 이하로 2개 이상의 같은 블록이 존재하면 이를 임시 List에 넣고, 이를 통한 매치 가능 여부를 판단, 이를 중복 없이 매치 가능한 블록들을 모은 List에 입력한다.


<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/2a8edbd5-8c7d-49c0-bc47-e61fe9907e0e" alt width=700>
<em></em>
</center>


이러한 과정을 통해 모은 List 중 하나를 선택해, 두 블록의 위치를 변경, swap하여 매치를  발생시킨다.



<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/1239a379-868a-46be-b696-8ffd73d4de72" alt width=700>
<em></em>
</center>



<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/106d6ef6-6aa9-4a98-a480-24e0456544a3" alt width=700>
<em></em>
</center>


그리고 swap이 이루어진 후에 update()에서 CreateGroup()을 호출, 매치가 이뤄진 블록들을 List에 입력, 제거하여 이러한 과정이 swap 횟수가 모두 떨어질 때까지 진행한다.


이를 활용, 지정된 목표 블록을 우선적으로 swap하는 자동 플레이를 구현하였다.

```c#
void Play(int idx)
{
	for(int i = 0; i < num_moves; i++)
	{
		for(each target in CurTargets)
		{
			if(!target.Value.Achieved)
			{
				Swap();
				updateState();
				break;
			}
		}
	}
	population[idx].fitness = remain_moves / num_moves;
}

void Swap()
{
	List<int> targetsInCell[]; 
	
	for(each c in Cells) // stage의 target들이 현재 grid에 있는지 확인하고, 존재한다면 position을 targetInCell에 추가한다. 
	{
		for(each target in CurTargets)
		{
			if(!target.Value.Achieved)
			{
				if(target.key == c.objectID) targetsInCell.Add({ c.Row, c.Column });
			}
		}
	}
	
	List<int> collectTargetMatch[]; //Matchable Blocks List에서 target을 포함한 Matchable Blocks의 idx를 저장 

	for(int i = 0;i < mgList.count; i++) 
	{
		for(int j = 0;j < targetsInCell.count; j++) 
		{
			bool isExist = false;
			
			//mgList[].Cell[2]는 est2로 match 시킬 위치의 바깥의 존재하는 block이다. match 시킬 위치의 블록이 아니므로 이를 제외하고 검사한다. 
			if (mgList[i].Cells[0].Row == targetsInCell[j].Row && mgList[i].Cells[0].Col == targetsInCell[j].Col) isExist = true; break;
			if (mgList[i].Cells[1].Row == targetsInCell[j].Row && mgList[i].Cells[1].Col == targetsInCell[j].Col) isExist = true; break;
			if (isExist) collectTargetMatch.Add(i);
		}
	}
	
	if(collectTargetMatch.Count == 0) // mgList에 target이 존재하지 않으면 mgList에서 아무거나 swap 
	{
		int swap_position_in_mgList = Random.Range(0, mgList.Count - 1);
		mgList[swap_position_in_mgList].SwapEstimate(); 
	}

	else  
	{
		int rand_num = Random.Range(0, 9); //target swap 확률 부여 
		
		if(rand_num > 2) // mgList에 target이 존재하면 collectTargetMatch 범위 안에서 하나의 idx를 선택해 이에 해당하는 mgList에서 swap 
		{
			int swap_position_in_collectTargetMatch = Random.Range(0, collectTargetMatch.Count - 1); 
			mgList[collectTargetMatch[swap_position_in_collectTargetMatch]].SwapEstimate();
		}
		else
		{
			int swap_position_in_mgList = Random.Range(0, mgList.Count - 1);
			mgList[rand_num].SwapEstimate(); 
		}
	}
}

void updateState()
{	
	CreateGroup(0); // match group을 생성해서 match된 block 확인 
	while(mgList.size() != 0)
	{
		if (CurrentGrid.GetFreeCells(true).Count > 0)
		{
			FillGridByStep()
			{
				// match block 아래에 빈 cell이 있다면 그 cell 위의 match block들을 y축 1만큼씩 이동.
				// match block 아래에 빈 cell이 없을 때까지 반복.
				// 작업이 끝난 후 남은 빈 cell에 random match block을 spawn.
			}
		}
		else
		{
			Collect();  // match된 블록을 수집한다.
			Destroy();  // Collect()에서 수집된 block들을 빈 cell로 변환하고 그 숫자만큰 score++한다.
		}
		CreateGroup(0); // 다시 한번 match 확인 
	}
}

```









































