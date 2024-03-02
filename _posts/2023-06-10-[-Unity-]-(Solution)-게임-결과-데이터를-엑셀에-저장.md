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


자동 플레이 시 게임이 종료되면, 사용자가 클리어에 필요했던 스왑 횟수가 나온다. 난이도 측정에 필요한 스왑 횟수를 저장하고, 반복 횟수에 따른 스왑 횟수의 평균을 구하기 위해, 이를 엑셀 파일로 저장하는 코드를 구현했다.


```c#

List<int> swapCntContainer = new List<int>();

private void Play()
{
	...

	// 게임 종료 시
	swapCntContainer.Add(스왑 횟수);
}

...
// 반복 종료 시
cs.swapContainer.Add(swapCntContainer);
```

이 외에도 플레이 가능한 맵이 세대 반복에 따라 어떻게 변하는 지, 적합도의 변화 등의 다양한 정보를 저장하였다.


```c#

using Mkey;
using System;
using System.Collections.Generic;
using System.IO;
using System.Text;
using UnityEngine;

public class CSVFileWriter : MonoBehaviour
{

    public List<string[]> data;

    public List<int> targetNeedCnt;

    public List<int> infeasiblePopulationCnt;
    public List<int> feasiblePopulationCnt;

    public List<int> curGenerationBestMean;
    public List<int> curGenerationBestStd;
    public List<int> curGenerationBestFitness;

    public List<int> bestMeanMove;
    public List<int> bestStd;
    public List<int> bestFitness;

    public List<List<int>> bestSwapContainer;
    public List<List<int>> swapContainer;
    public List<List<int>> matchContainer;

    public CSVFileWriter(Match3Helper m3h)
    {
        data = new List<string[]>();
        targetNeedCnt = new List<int>();
        foreach (var item in m3h.curTargets)
        {
            targetNeedCnt.Add(item.Value.ID);
            targetNeedCnt.Add(item.Value.NeedCount);
        }

        infeasiblePopulationCnt = new List<int>();
        feasiblePopulationCnt = new List<int>();

        curGenerationBestMean = new List<int>();
        curGenerationBestStd = new List<int>();
        curGenerationBestFitness = new List<int>();

        bestMeanMove = new List<int>();
        bestStd = new List<int>();
        bestFitness = new List<int>();

        bestSwapContainer = new List<List<int>>();
        swapContainer = new List<List<int>>();
        matchContainer = new List<List<int>>();

    }

    public void write(GeneticAlgorithm<char> ga, Match3Helper m3h)
    {
        string[] tempData = new string[1000];
        string[] gaDatas = new string[1000];

        gaDatas[0] = "Limit";
        gaDatas[1] = "populationSize";
        gaDatas[2] = ga.populationSize.ToString();
        gaDatas[3] = "elitism";
        gaDatas[4] = ga.elitism.ToString();
        gaDatas[5] = "mutationRate";
        gaDatas[6] = ga.mutationRate.ToString();

        gaDatas[7] = "generationLimit";
        gaDatas[8] = m3h.limits.generation.ToString();
        gaDatas[9] = "moveLimit";
        gaDatas[10] = m3h.limits.move.ToString();
        gaDatas[11] = "repeat";
        gaDatas[12] = m3h.limits.repeat.ToString();

        gaDatas[13] = "INPUT";
        gaDatas[14] = "gridRowSize";
        gaDatas[15] = m3h.rowSize.ToString();
        gaDatas[16] = "gridColSize";
        gaDatas[17] = m3h.colSize.ToString();

        gaDatas[18] = "targetMove";
        gaDatas[19] = ga.targetMove.ToString();
        gaDatas[20] = "targetStd";
        gaDatas[21] = ga.targetStd.ToString();
        gaDatas[22] = "targetFitness";
        gaDatas[23] = ga.targetFitness.ToString();
        gaDatas[24] = "targetCnt";

        int csvIdx = 25;

        gaDatas[csvIdx] = "map"; csvIdx++;
        gaDatas[csvIdx] = ga.population[0].allPottential.map.ToString(); csvIdx++;

        gaDatas[csvIdx] = "obstacle"; csvIdx++;
        gaDatas[csvIdx] = ga.population[0].allPottential.obstacle.ToString(); csvIdx++;

        gaDatas[csvIdx] = "blocked1"; csvIdx++;
        gaDatas[csvIdx] = ga.population[0].allPottential.blocked1.ToString(); csvIdx++;

        gaDatas[csvIdx] = "blocked2"; csvIdx++;
        gaDatas[csvIdx] = ga.population[0].allPottential.blocked2.ToString(); csvIdx++;

        gaDatas[csvIdx] = "blocked3"; csvIdx++;
        gaDatas[csvIdx] = ga.population[0].allPottential.blocked3.ToString(); csvIdx++;

        gaDatas[csvIdx] = "blocked4"; csvIdx++;
        gaDatas[csvIdx] = ga.population[0].allPottential.blocked4.ToString(); csvIdx++;

        gaDatas[csvIdx] = "overlay1"; csvIdx++;
        gaDatas[csvIdx] = ga.population[0].allPottential.overlay1.ToString(); csvIdx++;

        gaDatas[csvIdx] = "overlay2"; csvIdx++;
        gaDatas[csvIdx] = ga.population[0].allPottential.overlay2.ToString(); csvIdx++;

        gaDatas[csvIdx] = "overlay3"; csvIdx++;
        gaDatas[csvIdx] = ga.population[0].allPottential.overlay3.ToString(); csvIdx++;

        gaDatas[csvIdx] = "overlay4"; csvIdx++;
        gaDatas[csvIdx] = ga.population[0].allPottential.overlay4.ToString(); csvIdx++;

        gaDatas[csvIdx] = "somethingWrong"; csvIdx++;
        gaDatas[csvIdx] = ga.population[0].allPottential.somethingWrong.ToString(); csvIdx++;

        gaDatas[csvIdx] = "numOfMatchBlock"; csvIdx++;
        gaDatas[csvIdx] = m3h.numOfMatchBlock.ToString(); csvIdx++;

        string s111 = "'";
        gaDatas[csvIdx] = "gridObjects"; csvIdx++;
        for (int i = 0; i < ga.population[0].gridObjects.Count; i++)
        {
            s111 += ga.population[0].gridObjects[i].ToString();
        }
        gaDatas[csvIdx] = s111; csvIdx++;

        gaDatas[csvIdx] = "possibleCounting"; csvIdx++;
        for (int i = 0; i < ga.population[0].mapMatchPotentialList.Count; i++)
        {
            gaDatas[csvIdx] = ga.population[0].mapMatchPotentialList[i].ToString();
            csvIdx++;
        }

        string s = "'";

        for (int i = 0; i < m3h.gridSize; i++) s += ga.population[0].genes[i];

        gaDatas[csvIdx] = s;
        csvIdx++;

        int idx = 0;
        for (int i = 0; i < m3h.limits.generation; i++)
        {
            tempData = new string[1000];

            if (i < gaDatas.Length) tempData[0] = gaDatas[i];

            if (i < m3h.limits.generation)
            {
                tempData[1] = (i + 1).ToString();
                tempData[2] = 0.ToString();
                tempData[3] = 0.ToString();
                tempData[4] = 0.ToString();
                tempData[5] = 0.ToString();
                tempData[6] = 0.ToString();
                tempData[7] = 0.ToString();
                tempData[8] = 0.ToString();
                tempData[9] = 0.ToString();

                for (int j = 0; j < swapContainer[idx].Count; j++)
                {
                    tempData[10] += bestSwapContainer[i][j].ToString();
                    tempData[10] += ",";
                    tempData[11] += swapContainer[idx][j].ToString();
                    tempData[11] += ",";
                    tempData[12] += matchContainer[idx][j].ToString();
                    tempData[12] += ",";
                }
                idx++;
            }

            data.Add(tempData);
        }


        bool isFirst = true;
        while (idx < swapContainer.Count)
        {
            if (!isFirst)
            {
                if (idx < gaDatas.Length) tempData[0] = gaDatas[idx];
            }

            tempData = new string[1000];

            for (int j = 0; j < swapContainer[idx].Count; j++)
            {
                tempData[10 + ga.repeat + 1] += bestSwapContainer[idx][j].ToString();
                tempData[10 + ga.repeat + 1] += ",";
                tempData[11 + ga.repeat + 1] += swapContainer[idx][j].ToString();
                tempData[11 + ga.repeat + 1] += ",";
                tempData[12 + ga.repeat + 1] += matchContainer[idx][j].ToString();
                tempData[12 + ga.repeat + 1] += ",";
            }
            idx++;
            data.Add(tempData);

            isFirst = false;
        }

        if (data.Count < gaDatas.Length)
        {
            while (data.Count < gaDatas.Length)
            {
                tempData = new string[1000];
                tempData[0] = gaDatas[idx];
                idx++;
                data.Add(tempData);
            }
        }

        string[][] output = new string[data.Count][];

        for (int i = 0; i < output.Length; i++) output[i] = data[i];

        int length = output.GetLength(0);
        string delimiter = ",";

        StringBuilder sb = new StringBuilder();

        for (int index = 0; index < length; index++) sb.AppendLine(string.Join(delimiter, output[index]));


        string sPath = "/CSV/";

        sPath += m3h.csvFolder.ToString();
        sPath += '/';
        sPath += (m3h.csvCnt % 10).ToString();
        sPath += ".csv";
        m3h.csvCnt++;
        String filePath = Application.dataPath + sPath;

        StreamWriter outStream = System.IO.File.CreateText(filePath);

        outStream.WriteLine(sb);
        outStream.Close();
    }
}

```