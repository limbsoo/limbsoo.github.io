---
categories:
  - Coding test
  - 프로그래머스
tags:
  - 프로그래머스
  - codingTest
  - cpp
  - algorithm
---
# 문제
___

[https://school.programmers.co.kr/learn/courses/30/lessons/12951](https://school.programmers.co.kr/learn/courses/30/lessons/12951)

# 풀이
___

단순하게 아스키 코드로 풀어 공백에 따라 대문자와 소문자를 변환하여 string으로 리턴


```c++
#include <string>
#include <vector>

using namespace std;

string solution(string s) 
{
    string answer = "";

    for(int i = 0; i < s.size(); i++)
    {
        if (i == 0)
        {
            if (s[i] >= 'a' && s[i] <= 'z')
            {
                s[i] -= 32;
            }

        }

        else
        {
            if (s[i - 1] == ' ')
            {
                if (s[i] >= 'a' && s[i] <= 'z')
                {
                    s[i] -= 32;
                }
            }

            else
            {
                if (s[i] >= 'A' && s[i] <= 'Z')
                {
                    s[i] += 32;
                }
            }
        }
    }

    answer = s;

    return answer;
}

```



※ 다른 코드

공백에 따라 toupper(), tolower()함수를 사용하여 더 짧게 구현할 수 있다.

```c++
#include <string>
#include <vector>
using namespace std;

string solution(string s)
{
  string answer = "";
  answer += toupper(s[0]);
  for(int i = 1; i < s.size(); i++)
  {
    if(s[i-1] == ' ')
    { 
      answer += toupper(s[i]); // 공백 뒤에 대문자
    }
    
    else
    {
      answer += tolower(s[i]);
    }
  }
  return answer;
}

```
