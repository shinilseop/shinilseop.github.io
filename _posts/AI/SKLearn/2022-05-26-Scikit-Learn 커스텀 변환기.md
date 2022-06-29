---
title:  "Scikit Learn 커스텀 변환기"
excerpt: "Scikit Learn 커스텀 변환기 만드는법"

categories:
  - SKLearn
tags:
  - [Scikit-Learn, BaseEstimator, TransformerMixin, CombinedAttributesAdder]

toc: true
toc_sticky: true
 
date: 2022-05-26
last_modified_at: 2022-05-27

---

## 사이킷런 커스텀 변환기 만들기

사이킷 런에는 원하는 기능을 구현하는 변환기를 제작할 수 있다.

이는 BaseEstimator 와 TransformerMixin을 이용하여 제작하게 된다.

```
from sklearn.base import BaseEstimator, TransformerMixin

# 열 인덱스
rooms_ix, bedrooms_ix, population_ix, households_ix = 3, 4, 5, 6

class CombinedAttributesAdder(BaseEstimator, TransformerMixin):
    def __init__(self, add_bedrooms_per_room=True): # *args 또는 **kargs 없음
        self.add_bedrooms_per_room = add_bedrooms_per_room
    def fit(self, X, y=None):
        return self  # 아무것도 하지 않습니다
    def transform(self, X):
        rooms_per_household = X[:, rooms_ix] / X[:, households_ix]
        population_per_household = X[:, population_ix] / X[:, households_ix]
        if self.add_bedrooms_per_room:
            bedrooms_per_room = X[:, bedrooms_ix] / X[:, rooms_ix]
            return np.c_[X, rooms_per_household, population_per_household,
                         bedrooms_per_room]
        else:
            return np.c_[X, rooms_per_household, population_per_household]

attr_adder = CombinedAttributesAdder(add_bedrooms_per_room=False)
housing_extra_attribs = attr_adder.transform(housing.to_numpy())
```

위의 코드는 hands on machine learning 의 조합 특성을 추가하는 간단한 변환기의 예시이다.

캘리포니아의 집 가격 예측을 진행함에 있어서 특성들을 조합하여 새로운 특성을 구성해내는 변환기이다.

BaseEstimator와 TransformerMixin을 상속 받는 특정 클래스를 만든다.

Mixin이라는 이름은 파이썬에서 객체의 기능을 확장하려는 목적으로 만들어진 클래스를 뜻하는 말이다.

TransformerMixin 클래스는 fit\_transform() 메소드 하나를 가지고 있으며 이를 상속받는 모든 파이썬 클래스에 이 메소드를 제공한다.

TransformerMixin의 fit\_transform() 메소드는 fit()과 transform()을 단순히 연결한것이다.

fit\_transform() 메소드는 pipeline에 여러 변환기 혹은 추정기를 추가하여 사용할때 필요한 메소드이다.

pipeline에서 fit\_transform() 메소드를 출력하며, 해당 메소드가 없더라도 fit()과 transform() 메소드를 차례로 호출한다고 한다.