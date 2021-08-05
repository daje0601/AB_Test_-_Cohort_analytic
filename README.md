## 🔶시나리오

> myrealtrip의 데이터 분석가로 입사를 하게 되었습니다. 마케팅팀에서 웹페이지 변경 건 관련 A/B 테스트를 요청하셨습니다. 팀장님께 보고드린 후 해당 분석을 담당하게 되었습니다. google analytics(GA)를 활용하여 funnel 분석 결과, 장바구니에서 결제로 이어지는 구간에서 가장 큰 이탈이 있는 것으로 확인되었습니다.
현재 장바구니에서 결제로 전환되는 비율은 23%였으며, 마케팅팀과 협의하여 25%로 전환율을 올리면 해당 분석은 성공한 것으로 간주하기로 협의하였습니다.

## 🔶 A/B test 상세내역

- 목표 지수 : 전환율 2 % 상승 ( 장바구니 -> 결제 )
- 기간 : 21일

      ( 주중, 주말간의 고객의 행동 패턴을 모두 포함하여야 하기에 21일로 설정 )

- 가설 : 우리는 결제 페이지의 디자인을 변경하여 구매 전환율을 2% 이상 증가할 것이다.

      ( 신뢰도 : 95% / α:0.05 / One- and two-tailed tests 사용 : 전환율이 낮아질 수도 있음 )

- A/B테스트 내용 : 기존의 사이트와 버튼을 변경한 사이트 둘을 50%의 확률로 랜덤하게 노출시킴
- 지표 확인 : 21일 후 전환율 평균값 비교

(단, myrealtrip의 기존 데이터가 누적되어 있고 고객들의 행동패턴을 사전에 파악하고 있다고 가정하고 해당 분석에서는 A/A test는 진행하지 않도록 하겠습니다.)

![image](https://user-images.githubusercontent.com/73736988/128304732-fb082f36-0e91-4300-a9f5-9304c14c5532.png)


## 🔶 표본 설정 & 전환율

### 1. **표본 설정**

- 본격적인 분석에 앞서서 표본을 설정하는 작업이 필요합니다.
- 아무리 A/B테스트를 철저하게 하더라도 30명 미만으로 고객이 유입된다면 의미없는 분석이 됩니다.
- 또한, 표본의 크기는 p-value의 민감도에 영향을 주기 때문에 적정한 표본의 크기는 중요하다고 생각하였습니다. ( 표본 커질수록 p-value는 작아지기 때문입니다. )
- 표본의 크기는 검정력 분석(power of test)로 구할 수 있으며, 관례적으로 power를 0.8로 사용하기에 해당 프로젝트에서도 0.8로 활용하도록 하겠습니다.
- 또한, 한 유저는 한 그룹에만 들어가야하므로 중복된 유저는 제거해 주었습니다.

```jsx
import scipy.stats as stats 
import statsmodels.stats.api as sms 
from math import ceil 

# 예상 비율을 기반으로 effectsize(효과) 계산 
effect_size = sms.proportion_effectsize(0.23, 0.25) 

# 샘플 크기 구하기 
sample_size = sms.NormalIndPower().solve_power(effect_size, power=0.8, 
                                               alpha=0.05, ratio=1)

# 즉, A,B 그룹에 최소 7155명의 고객 수 필요 
print("sample_size : ", round(sample_size), "명") 
print("effect_size : ", effect_size.round(5))
```

### 2. **전환율**

```jsx
# 전환율을 계산해보겠습니다. 
conversion_rates = ab_test.groupby('group')['converted']

std_p = lambda x: np.std(x, ddof=0)              # Std. deviation of the proportion 구하기 
se_p = lambda x: stats.sem(x, ddof=0)            # Std. error of the proportion (std / sqrt(n)) 구하기 

conversion_rates = conversion_rates.agg([np.mean, std_p, se_p])
conversion_rates.columns = ['conversion_rate', 'std_deviation', 'std_error']

conversion_rates.style.format('{:.3f}')

```


## 🔶 A/B 테스트


### 1. **진행 절차**

- A, B group 생성
- **Shapiro-wilk test**를 활용한 정규성 확인
- Shapiro-wilk test를 통과하지 못한 경우, **Mann Whitney U Test** 두 그룹 차이 확인
- Shapiro-wilk test를 통과한 경우, **Levene test**로 두 그룹간의 분산 확인
- Levene test를 통과하지 못한 경우, **apply Welch test**로 평균 확인
- Levene test를 통과한 경우, **t-test**로 평균 확인 / [A/B 테스트 블로그 정리 링크](https://daje0601.tistory.com/267)

### 2. 코드

```jsx
# A/B Testing 함수
def AB_Test(dataframe, group, target):
    
    # 라이브러리 
    from scipy.stats import shapiro
    import scipy.stats as stats
    
    # 데이터 샘플링 
    groupA = dataframe[dataframe[group] == 'control'].sample(n=7200, random_state=42)[target]
    groupB = dataframe[dataframe[group] == 'treatment'].sample(n=7200, random_state=42)[target]

    # Normality(정규성) 확인하기 
    ntA = shapiro(groupA)[1] < 0.05
    ntB = shapiro(groupB)[1] < 0.05
    # H0(귀무가설): 정규분포이다. - 결과값 : False
    # H1(대립가설): 정규분포가 아니다. - 결과값 : True
    
    # "H0(귀무가설)이 성립할 경우, 데이터는 정규분포이므로 모수검정을 진행합니다. "
    if (ntA == False) & (ntB == False): 
        # 두 그룹간에 분산이 같은지를 확인합니다. 
        leveneTest = stats.levene(groupA, groupB)[1] < 0.05
        # H0(귀무가설): 두 그룹간의 분산이 같다. - 결과값 : False
        # H1(대립가설): 두 그룹간의 분산이 다르다. - 결과값 : True
        
        if leveneTest == False:
            # 분산이 같은 경우, t-test를 진행합니다. 
            ttest = stats.ttest_ind(groupA, groupB, equal_var=True)[1]
            # H0(귀무가설): 두 그룹간의 평균이 같다. - 결과값 : False
            # H1(대립가설): 두 그룹간의 평균이 다르다. - 결과값 : True
        else:
            # 분산이 다른 경우, leveneTest를 진행합니다. 
            ttest = stats.ttest_ind(groupA, groupB, equal_var=False)[1]
            # H0(귀무가설): 두 그룹간의 평균이 같다. - 결과값 : False
            # H1(대립가설): 두 그룹간의 평균이 다르다. - 결과값 : True
    else:
        # Non-Parametric Test
        ttest = stats.mannwhitneyu(groupA, groupB)[1] 
        # H0(귀무가설): 두 그룹이 같다. - 결과값 : False
        # H1(대립가설): 두 그룹이 다르다. - 결과값 : True
        
    # 결과 도출 
    temp = pd.DataFrame({
        "AB Hypothesis":[ttest < 0.05], 
        "p-value":[ttest]
    })
    temp["Test Type"] = np.where((ntA == False) & (ntB == False), "Parametric", "Non-Parametric")
    temp["AB Hypothesis"] = np.where(temp["AB Hypothesis"] == False, "Fail to Reject H0", "Reject H0")
    temp["Comment"] = np.where(temp["AB Hypothesis"] == "Fail to Reject H0", "A/B groups are similar!", "A/B groups are not similar!")
    
    # 컬럼 생성 
    if (ntA == False) & (ntB == False):
        temp["Homogeneity"] = np.where(leveneTest == False, "Yes", "No")
        temp = temp[["Test Type", "Homogeneity","AB Hypothesis", "p-value", "Comment"]]
    else:
        temp = temp[["Test Type","AB Hypothesis", "p-value", "Comment"]]
    
    # Print Hypothesis
    print("# A/B Testing Hypothesis")
    print("H0: A == B")
    print("H1: A != B", "\n")
    
    return temp
    

AB_Test(dataframe=ab_test, group = "group", target = "converted")
```

## 🔶 A/B 테스트


- 확인 결과, sharpio-wilk test 결과가 정규분포를 따르지 않아 Non-Parametric 검정을 통하여 A/B test가 진행되었습니다.
- p-value(0.3585)가 α(0.05)보다 크므로 귀무가설을 기각할 수 없습니다.
- 이는 A그룹과 B그룹은 유사한 것으로 통계적 확인이 되었고, 저희가 변경한 웹 디자인은 유효하지 않은 것으로 판명되었습니다.
- 이에, 빠르게 다른 가설을 설정하여 실험을 진행할 필요가 있습니다.

![image](https://user-images.githubusercontent.com/73736988/128304833-f3adcb3a-f8d6-4a4b-8bc4-68d880dd0307.png)
