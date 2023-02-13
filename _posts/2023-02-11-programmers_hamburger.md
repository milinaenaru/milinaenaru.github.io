---
layout: single
title: "햄버거 만들기 programmers 문제 풀이"
---



프로그래머스 lv1. 햄버거 만들기 제출 답안

```c
#include <stdio.h>
#include <stdbool.h>
#include <stdlib.h>

// ingredient_len은 배열 ingredient의 길이입니다.
int solution(int ingredient[], size_t ingredient_len) {
    int answer = 0;
    int tmpLen = ingredient_len;
    int tmp;
    for(int i = 0; i < tmpLen - 3; i ++)
    {
        if((ingredient[i] == 1) && (ingredient[i+1] == 2) && 
		   (ingredient[i+2] == 3) && (ingredient[i+3] == 1))
        {
            answer += 1;
            for(int j = i; j < tmpLen; j++)
            {
                if(j + 4 < tmpLen)
                    ingredient[j] = ingredient[j+4];
                else
                    ingredient[j] = 0;
            }
            tmpLen -= 4;
            for(int j = i-1; j >= 0; j--)
            {
                if(ingredient[j] == 1)
                {
                    i = j;
                    break;
                }
            }
            i--;
        }
    }
    if(answer != 0)
        return answer;
    else
    {
        printf("상수가 포장할 수 있는 햄버거가 없습니다.");
        return 0;
    }
}
```

