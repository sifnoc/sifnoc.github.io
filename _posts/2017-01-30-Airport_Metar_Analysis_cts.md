---
layout: post
title: "공항기상정보(METAR) 데이터 분석"
---


 
## metar 데이터 파싱 그리고 데이터프레임 차트화한 인사이트 

항공기에서 조종사는 목적지 공항의 기상정보를 수시로 받게 됩니다. 기상정보는 METAR의 형식을 취하고 있고 이는 목적지 공항의 기상관측소에서
발부하고 있는 기상 실황입니다. 조종사는 METAR를 보고 최종적으로 활주로에 접근하기전에 도착지 기상이 어떨지 생각하게 됩니다.

항공기를 운영하는 항공사 입장에서는 출발전에 어떠한 기상이 예상되는지는 도착지 공항에서 발부하고 있는 TAF(Terminal aerodrome
forecast) 정보를 통해서 파악하고 있습니다. 만약에 TAF상 기상이 좋지 않을것으로 예상되면 지연을 하거나 결항을 결정하게 됩니다.

* TAF는 공항의 기상예보, METAR는 공항의 기상실황

저는 항공사에서 취항지(도착지) 및 교체공항(도착지 공항에 못내렸을 경우 회항하는 공항)에 대한 모든 기상정보를 수집하여 분석하는 일을 하고
있습니다. TAF 와 METAR 기상정보가 작성되는 기준이 있으며 이는 ICAO ANNEX3에 근거하여 작성되도록 권장하고 있습니다. 그래서 꼭
지켜야 할 의무사항이 아니라서 안지켜지기도 하지만요, 여하간 항공기 운항에 중요한 정보로는 METAR 와 TAF가 있는데 이번에는 METAR를
분석해보기로 하였습니다. TAF의 경우 구문이 더 까다로워서 파싱하는데 더 어려워서 시도를 아직 해보지 않았습니다.

물론, Perl로 되어 있는 TAF와 METAR 파싱 스크립트가 있습니다 <a
href="http://metaf2xml.sourceforge.net/">metaf2xml</a>. 하지만 저는 파이썬으로 구현해보고
싶습니다(저는 perl을 모르니까요..). 그리고 파싱된 데이터를 분석해보겠습니다.

#### METAR 예시

<table>
<thead>
<tr>
<th>공항명</th>
<th>종류</th>
<th>METAR(UTC)</th>
</tr>
</thead>

<tr>
<th class="stn">인천(RKSI)</th>
<td class="type">METAR</td>
<td class="msg">METAR RKSI 251730Z 13003KT 7000 NSC M05/M08 Q1032 NOSIG=</td>
</tr>

<tr>
<th class="stn">김포(RKSS)</th>
<td class="type">METAR</td>
<td class="msg">METAR RKSS 251700Z 32001KT 7000 NSC M11/M14 Q1032 NOSIG=</td>
</tr>

<tr>
<th class="stn">제주(RKPC)</th>
<td class="type">METAR</td>
<td class="msg">METAR RKPC 251700Z 35003KT 310V060 9999 SCT030 03/M04 Q1033
NOSIG=</td>
</tr>

<tr>
<th class="stn">울산(RKPU)</th>
<td class="type">METAR</td>
<td class="msg">METAR RKPU 251700Z AUTO 35011KT 9999  NCD M02/M13 Q1030=</td>
</tr>

<tr>
<th class="stn">무안(RKJB)</th>
<td class="type">METAR</td>
<td class="msg">METAR RKJB 251700Z AUTO 07005KT 9999  NCD M04/M06 Q1033=</td>
</tr>

<tr>
<th class="stn">여수(RKJY)</th>
<td class="type">METAR</td>
<td class="msg">METAR RKJY 251700Z AUTO 31009KT 9999  NCD M02/M09 Q1031=</td>
</tr>
</table>

* 출처 : <a href="kama.kma.go.kr">항공기상청</a>


METAR 형식은 위와 같습니다. METAR 구문해석은 다루지 않겠습니다.
항공기 운항에 중요한 요소는 바람, 시정 그리고 운고가 되겠습니다.

데이터 소스는 <a href = "http://www.ogimet.com/index.phtml.en">OGIMET</a>에서 가져왔습니다. 한번
리퀘스트에 31일 제한이 있지만 한번에 리퀘스트할 수 있는 공항갯수에서는 제한이 없어서 아시아나가 취항하고 있는 공항에 대해서 한꺼번에
가져왔습니다. 서버로 블럭은 안당했지만 모두 크롤링하는데 상당히 걸렸습니다(저장한 Text파일만 1.6기가 였습니다).

항공기 운항에 중요한 요소가 바람, 시정, 운고라고 말씀드렸는데요 이걸 text에서 분리해내야 합니다. 정규식을 써서 끄집어 냈는데요 공항마다
METAR 스타일이 달라서 바꿔줘야 합니다.(왜 ICAO ANNEX 3를 제대로 지키지 않는가!)

다음달 출장이 계획된 일본 북해도 치토세(CTS)공항의 METAR를 분석해보겠습니다. 



{% highlight python %}
#사용된 정규식
metar = re.compile('(\d{{12}}\sMETAR\s{apo}\s\d{{6}}Z.*=)\\s'.format(apo="RJCC"))
speci = re.compile('(\d{{12}}\sSPECI\s{apo}\s\d{{6}}Z.*=)\\s'.format(apo="RJCC"))

wind_comp = re.compile(r'(\d{3}|VRB)(\d{1,3}){1}G{0,1}(\d{2,3}){0,1}(KT|MPS)')
vis_comp = re.compile(r'KT\s(\d{4}|CAVOK)\s')
cig_comp = re.compile(r'\s(BKN|OVC|VV)(\d{3})(CU|CB|TC|TCU|CI){0,1}')
wx_SN_comp = re.compile(r'(\-SN|\+SN|SN)')
wx_TS_comp = re.compile(r'(\-TS|\+TS|TS)')
wx_FG_comp = re.compile(r'FG')
qnh_comp = re.compile(r'\sA(\d+)')
temp_comp = re.compile(r'\s(M{0,1})(\d+)/(M{0,1})(\d+)\s') 
{% endhighlight %}

 
metar, speci가 있는데, 공항 기상관측소에서 정기적으로 관측하는 기상정보는 METAR로, 비정기적으로 관측하는 기상정보는 SPECI로 시작됩니다. SPECI 발표 기준은 풍향, 풍속, 시정, 운고에서 기준값이상 변화시 관측하여 발표 됩니다. (나라마다 혹은 관측소마다 미묘한 차이를 가지고 있습니다.)

기상현상군을 SN, TS, FG만 가져오도록 하였습니다. 파싱하여 만든 DataFrame 샘플은 아래와 같습니다. 







<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>W_DIR</th>
      <th>W_SPD</th>
      <th>W_GST</th>
      <th>VIS</th>
      <th>CIG</th>
      <th>WX</th>
      <th>QNH</th>
      <th>TEMP</th>
      <th>DEWP</th>
      <th>Raw_Metar</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>2016-05-22 03:00:00</th>
      <td>170</td>
      <td>11</td>
      <td>0</td>
      <td>9999</td>
      <td>200</td>
      <td>None</td>
      <td>2998.0</td>
      <td>23.0</td>
      <td>14.0</td>
      <td>220300Z 17011KT 9999 FEW030 BKN/// 23/14 Q101...</td>
    </tr>
    <tr>
      <th>2016-05-22 02:30:00</th>
      <td>170</td>
      <td>11</td>
      <td>0</td>
      <td>9999</td>
      <td>200</td>
      <td>None</td>
      <td>2999.0</td>
      <td>24.0</td>
      <td>14.0</td>
      <td>220230Z 17011KT 9999 FEW015 BKN/// 24/14 Q101...</td>
    </tr>
    <tr>
      <th>2016-05-22 02:00:00</th>
      <td>180</td>
      <td>11</td>
      <td>0</td>
      <td>9999</td>
      <td>200</td>
      <td>None</td>
      <td>3000.0</td>
      <td>23.0</td>
      <td>13.0</td>
      <td>220200Z 18011KT 9999 FEW010 BKN/// 23/13 Q101...</td>
    </tr>
    <tr>
      <th>2016-05-22 01:30:00</th>
      <td>190</td>
      <td>10</td>
      <td>0</td>
      <td>9999</td>
      <td>200</td>
      <td>None</td>
      <td>3001.0</td>
      <td>21.0</td>
      <td>13.0</td>
      <td>220130Z 19010KT 9999 FEW010 SCT/// 21/13 Q101...</td>
    </tr>
    <tr>
      <th>2016-05-22 01:00:00</th>
      <td>150</td>
      <td>11</td>
      <td>0</td>
      <td>9999</td>
      <td>200</td>
      <td>None</td>
      <td>3000.0</td>
      <td>20.0</td>
      <td>13.0</td>
      <td>220100Z 15011KT 9999 FEW008 SCT/// 20/13 Q101...</td>
    </tr>
  </tbody>
</table>
</div>


 
시간대를 살펴보면 SPECI때문에 정시관측에 포함되지 않은 데이터들이 있습니다.<br>
SPECI 관측데이터는 대부분 악기상 정보를 포함하고 있기 모두 챙겨서 분석에 들어가봅시다. 



{% highlight python %}
df_met['Times'] = df_met.index.map(lambda x : x.strftime("%H:%M"))
print(df_met['Times'].unique()) 
{% endhighlight %}


    ['03:00' '02:30' '02:00' ..., '06:58' '03:29' '23:57']

 
TimeRound 처리를 위한 Help Function을 만들어서 적용해줘야 겠습니다. 



{% highlight python %}
def Timeround(dt):
    if dt.minute < 15:
        return dt.replace(minute = 0)
    elif dt.minute >= 15 and dt.minute < 45:
        return dt.replace(minute = 30)
    elif dt.minute >= 45:
        dt = dt + timedelta(hours = 1)
        return dt.replace(minute = 0) 
{% endhighlight %}

 
30분 단위 발표되는 정시발표 METAR 시간대로 모두 포함되도록 하였습니다. 



{% highlight python %}
df_met['Times'] = df_met.index.map(lambda x : Timeround(x).strftime("%H:%M"))
print(np.sort(df_met['Times'].unique())) 
{% endhighlight %}


    ['00:00' '00:30' '01:00' '01:30' '02:00' '02:30' '03:00' '03:30' '04:00'
     '04:30' '05:00' '05:30' '06:00' '06:30' '07:00' '07:30' '08:00' '08:30'
     '09:00' '09:30' '10:00' '10:30' '11:00' '11:30' '12:00' '12:30' '13:00'
     '13:30' '14:00' '14:30' '15:00' '15:30' '16:00' '16:30' '17:00' '17:30'
     '18:00' '18:30' '19:00' '19:30' '20:00' '20:30' '21:00' '21:30' '22:00'
     '22:30' '23:00' '23:30']

 
2005년 1월 1일 부터 2016년 5월 22일까지 158,004개의 METAR 데이터를 얻었습니다.

데이터프레임에서 동계와 하계 두 개로 데이터프레임을 재생성하여 진행하였습니다.
(지금은 이렇게 테이블을 재생성 하지 않습니다..)

기본적으로 일변화를 누적하여 비교하려고 하기 때문에 Hour로 Groupby 시켰습니다. 



{% highlight python %}
df_winter = df_met[(df_met.index.month >= 11 ) | (df_met.index.month <= 3)]
df_summer = df_met[(df_met.index.month > 3 ) & (df_met.index.month < 11)]

df_winter_gb = df_winter.groupby('Times')
df_summer_gb = df_summer.groupby('Times') 
{% endhighlight %}







 
시정부분은 크게 건드릴 필요가 없는것 같습니다. 



{% highlight python %}
np.sort(df_met['VIS'].unique())[::-1] 
{% endhighlight %}





array([9999, 9000, 8000, 7000, 6000, 5000, 4900, 4800, 4700, 4500, 4300,
       4200, 4000, 3800, 3500, 3400, 3300, 3200, 3100, 3000, 2800, 2700,
       2600, 2500, 2400, 2300, 2200, 2100, 2000, 1900, 1800, 1700, 1600,
       1500, 1400, 1300, 1200, 1100, 1000,  900,  800,  700,  600,  500,
        400,  300,  200,  100]) 


 
시간대별 시정분포도를 Heatmap으로 그려보기 위한 Empty 테이블을 만들어봅니다. 



{% highlight python %}
vis_spread = pd.DataFrame(index=np.sort(df_met['VIS'].unique())[::-1],
                          columns=np.sort(df_met['Times'].unique()))
vis_spread = vis_spread.fillna(0)
vis_spread.head() 
{% endhighlight %}





<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>00:00</th>
      <th>00:30</th>
      <th>01:00</th>
      <th>01:30</th>
      <th>02:00</th>
      <th>02:30</th>
      <th>03:00</th>
      <th>03:30</th>
      <th>04:00</th>
      <th>04:30</th>
      <th>...</th>
      <th>19:00</th>
      <th>19:30</th>
      <th>20:00</th>
      <th>20:30</th>
      <th>21:00</th>
      <th>21:30</th>
      <th>22:00</th>
      <th>22:30</th>
      <th>23:00</th>
      <th>23:30</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>9999</th>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>...</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>9000</th>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>...</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>8000</th>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>...</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>7000</th>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>...</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>6000</th>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>...</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
    </tr>
  </tbody>
</table>
<p>5 rows × 48 columns</p>
</div>


 
위와 같이 최종적으로 얻을 result dataframe을 미리 만들어 보는 방법은 제가 Pandas를 처음익히면서 사용한 방법으로 이해하기
쉬운 접근법입니다,<br>
후에 메모리나 속도를 위해서 가급적이면 groupby를 활용해서 분석하는것이 좋습니다.
(지금은 groupby로 하고 있습니다.) 
 
### RJCC ( CTS 치토세 ) 공항 시정 분석

앞서 만들어 놓은 vis_spread에 값을 채워보도록 하겠습니다.<br> 






{% highlight python %}
for i in vis_spread.columns:
    df_temp = df_winter_gb.get_group(i)
    vis_counts = dict(df_temp['VIS'].value_counts())
    for j in vis_spread.index:
        vis_spread[i][j] = vis_counts.get(j) 
{% endhighlight %}






![png]({{ '/assets/' | prepend: site.url }}/rjcc_metar_analysis_files/rjcc_metar_analysis_23_0.png)

 
일출 시간대(2000~2200Z) 가장 300m 미만 저시정 비율이 높은 경향을 보이는데, 안개의 영향이 가장큰 요인으로 생각됩니다. 

![png]({{ '/assets/' | prepend: site.url }}/rjcc_metar_analysis_files/rjcc_metar_analysis_25_1.png)

 
2000m 이하에서의 시정관측 횟수를 HeatMap으로 그려보면 위와 같습니다.
시정값으로 많은 비중을 보이는부분이 900m 부근과 1600m 에 몰려 있습니다.
이유인즉슨, 1mile = 1600m, 1/2mile = 800m 로 관측 및 예보에 사용되는 경계값이기 때문입니다.

그러니 관측자 입장에서는 시정경계값을 넘지 않는선에서 관측하는 경향을 보여주는 것이지요.

이번에는 하계 시정관측값에 대해서 분석해 보겠습니다. 








![png]({{ '/assets/' | prepend: site.url }}/rjcc_metar_analysis_files/rjcc_metar_analysis_28_0.png)

 
동계 저시정분포와 확연하게 시간대별 시정분포도가 다름을 보실 수가 있습니다.
주로 야간시간대에 낮은 저시정 비율을 나타내는데, 여기서 기상현상 비율을 그려보겠습니다.

특히 13:30Z에 비중이 높은데 확인해봐야할 부분입니다. 왜그럴까요? 이 부분에 대해서는 나중에 다시한번 들여다 보아야 겠습니다.<br>

SN, FG, TS 기상현상군의 횟수를 카운트하여 차트로 비교해보도록 하겠습니다. 



{% highlight python %}
df_summer['WX'].unique() 
{% endhighlight %}





array([None, 'FG', 'SN', '-TS', 'TS', '+TS', '-SN'], dtype=object) 


 
METAR에 관측된 기상군데 접두어(-, +) 가 붙어 있기때문에 단일화 시켜주겠습니다. 



{% highlight python %}
df_summer['WX_i'] = df_summer['WX'].apply(lambda x: x[-2:] if x != None else x)
df_winter['WX_i'] = df_winter['WX'].apply(lambda x: x[-2:] if x != None else x)

df_summer['WX_i'].unique() 
{% endhighlight %}





array([None, 'FG', 'SN', 'TS'], dtype=object) 


 
시정이 800m 미만이면서 SN, FG, TS 이 관측되었을때가 얼마나되는지 어떤지 그려보겠습니다. 



{% highlight python %}
df_wx_gb = df_summer[df_summer['VIS'] < 800].groupby(['Times', 'WX_i'])
df_wx = pd.DataFrame(df_wx_gb.count()['WX'].reset_index())
df_wx.rename(columns={'WX_i':'WX', 'WX':'Count'}, inplace=True)

sns.set(style="whitegrid", font='NanumGothic')

g = sns.factorplot(x="Times", y="Count", hue="WX", data=df_wx,
                   size=10, kind="bar", palette="muted")
plt.xticks(rotation=45)
g.set(ylim=(0, 200))
g.despine(left=True)
g.set_ylabels("기상현상 관측 횟수")
g.set_xlabels("UTC Times") 
{% endhighlight %}





<a></a> 




![png]({{ '/assets/' | prepend: site.url }}/rjcc_metar_analysis_files/rjcc_metar_analysis_34_1.png)

 
역시 하계(4월~10월)은 안개로 인한 시정저하가 현저해 보입니다.

다음은 동계(11월~3월)에 저시정을 일으켰던 기상군을 보겠습니다. 



{% highlight python %}
df_wx_gb = df_winter[df_winter['VIS'] < 800].groupby(['Times', 'WX_i'])
df_wx = pd.DataFrame(df_wx_gb.count()['WX'].reset_index())
df_wx.rename(columns={'WX_i':'WX', 'WX':'Count'}, inplace=True)

sns.set(style="whitegrid", font='NanumGothic')

g = sns.factorplot(x="Times", y="Count", hue="WX", data=df_wx,
                   size=10, kind="bar", palette="muted")
plt.xticks(rotation=50)
g.set(ylim=(0, 100))
g.despine(left=True)
g.set_ylabels("기상현상 관측 횟수")
g.set_xlabels("UTC Times") 
{% endhighlight %}





![png]({{ '/assets/' | prepend: site.url }}/rjcc_metar_analysis_files/rjcc_metar_analysis_36_1.png)

 
동계 저시정 기상현상분포의 다른점은 확연하게 강설(SN)로 인한 저시정(800m 미만)일때와 안개(FG)로 인한 비율이 비슷해보입니다.<br>
다만 강설은 주로 낮시간대, 안개는 주로 야간시간대 발생하는 것을 볼 수 있습니다.<br>
<br>다음은 HEATMAP으로 그려본 시정(Visibility) 일변화 입니다.<br>

![png]({{ '/assets/' | prepend: site.url }}/rjcc_metar_analysis_files/rjcc_metar_analysis_39_1.png)

 
동계 시정분포도는 강설(SN)로 인한 시정저하로 인해서 낮시간대에 저시정 분포가 높은 반면,
하계 시정분포도는 안개(FG)로 인한 시정저하로 인해서 야간시간대에 저시정 분포가 높은 경향을 볼수 있습니다. 
 
### RJCC ( CTS 치토세 ) 공항 저운고 분석

이번에는 Low Ceiling(저운고) 분포에 대해서 분석해보겠습니다.
보통 운고(Ceiling)이라고 하면 METAR의 구름부분에 Broken이상의 차폐되는 고도를 말합니다.
예를들어, BKN005 면 500FT 운고가 되는 것입니다.

항공기 운항결정에도 중요한 요소로, 항공기 접근절차상 제한치와 연결되어 있는 부분입니다.
조종사 입장에서는 Runway Insight되는 고도와 비슷할 수 있습니다.

항공기 접근절차마다 다르지만, Jepssen 이나 AIP 차트상 Landing Minima의 DA(H) 확인해보면
대체로 CAT1은 200ft,  CAT2는 100ft로 나와 있기때문에. 이를 기준을 잡고 분석해보도록 하겠습니다. 









{% highlight python %}
cig_spread.tail(10) 
{% endhighlight %}





<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>00:00</th>
      <th>00:30</th>
      <th>01:00</th>
      <th>01:30</th>
      <th>02:00</th>
      <th>02:30</th>
      <th>03:00</th>
      <th>03:30</th>
      <th>04:00</th>
      <th>04:30</th>
      <th>...</th>
      <th>19:00</th>
      <th>19:30</th>
      <th>20:00</th>
      <th>20:30</th>
      <th>21:00</th>
      <th>21:30</th>
      <th>22:00</th>
      <th>22:30</th>
      <th>23:00</th>
      <th>23:30</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>9</th>
      <td>9.0</td>
      <td>21.0</td>
      <td>12.0</td>
      <td>31.0</td>
      <td>12.0</td>
      <td>16.0</td>
      <td>18.0</td>
      <td>21.0</td>
      <td>14.0</td>
      <td>28.0</td>
      <td>...</td>
      <td>6.0</td>
      <td>8.0</td>
      <td>9.0</td>
      <td>14.0</td>
      <td>8.0</td>
      <td>22.0</td>
      <td>12.0</td>
      <td>18.0</td>
      <td>21.0</td>
      <td>28.0</td>
    </tr>
    <tr>
      <th>8</th>
      <td>16.0</td>
      <td>17.0</td>
      <td>23.0</td>
      <td>27.0</td>
      <td>24.0</td>
      <td>32.0</td>
      <td>27.0</td>
      <td>23.0</td>
      <td>29.0</td>
      <td>29.0</td>
      <td>...</td>
      <td>9.0</td>
      <td>6.0</td>
      <td>7.0</td>
      <td>7.0</td>
      <td>7.0</td>
      <td>11.0</td>
      <td>17.0</td>
      <td>18.0</td>
      <td>18.0</td>
      <td>25.0</td>
    </tr>
    <tr>
      <th>7</th>
      <td>7.0</td>
      <td>6.0</td>
      <td>14.0</td>
      <td>6.0</td>
      <td>9.0</td>
      <td>13.0</td>
      <td>5.0</td>
      <td>10.0</td>
      <td>15.0</td>
      <td>23.0</td>
      <td>...</td>
      <td>8.0</td>
      <td>6.0</td>
      <td>10.0</td>
      <td>10.0</td>
      <td>8.0</td>
      <td>11.0</td>
      <td>16.0</td>
      <td>19.0</td>
      <td>6.0</td>
      <td>5.0</td>
    </tr>
    <tr>
      <th>6</th>
      <td>27.0</td>
      <td>24.0</td>
      <td>22.0</td>
      <td>20.0</td>
      <td>10.0</td>
      <td>40.0</td>
      <td>28.0</td>
      <td>37.0</td>
      <td>24.0</td>
      <td>27.0</td>
      <td>...</td>
      <td>15.0</td>
      <td>12.0</td>
      <td>22.0</td>
      <td>27.0</td>
      <td>14.0</td>
      <td>12.0</td>
      <td>16.0</td>
      <td>39.0</td>
      <td>32.0</td>
      <td>29.0</td>
    </tr>
    <tr>
      <th>5</th>
      <td>22.0</td>
      <td>50.0</td>
      <td>41.0</td>
      <td>47.0</td>
      <td>44.0</td>
      <td>32.0</td>
      <td>32.0</td>
      <td>34.0</td>
      <td>32.0</td>
      <td>52.0</td>
      <td>...</td>
      <td>21.0</td>
      <td>17.0</td>
      <td>20.0</td>
      <td>36.0</td>
      <td>37.0</td>
      <td>44.0</td>
      <td>45.0</td>
      <td>44.0</td>
      <td>40.0</td>
      <td>46.0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>24.0</td>
      <td>38.0</td>
      <td>24.0</td>
      <td>37.0</td>
      <td>28.0</td>
      <td>31.0</td>
      <td>25.0</td>
      <td>30.0</td>
      <td>35.0</td>
      <td>38.0</td>
      <td>...</td>
      <td>16.0</td>
      <td>13.0</td>
      <td>19.0</td>
      <td>26.0</td>
      <td>15.0</td>
      <td>16.0</td>
      <td>18.0</td>
      <td>27.0</td>
      <td>31.0</td>
      <td>23.0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>35.0</td>
      <td>35.0</td>
      <td>14.0</td>
      <td>10.0</td>
      <td>19.0</td>
      <td>13.0</td>
      <td>13.0</td>
      <td>11.0</td>
      <td>13.0</td>
      <td>13.0</td>
      <td>...</td>
      <td>13.0</td>
      <td>11.0</td>
      <td>14.0</td>
      <td>19.0</td>
      <td>21.0</td>
      <td>19.0</td>
      <td>18.0</td>
      <td>25.0</td>
      <td>24.0</td>
      <td>42.0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>16.0</td>
      <td>6.0</td>
      <td>10.0</td>
      <td>5.0</td>
      <td>9.0</td>
      <td>16.0</td>
      <td>2.0</td>
      <td>4.0</td>
      <td>10.0</td>
      <td>10.0</td>
      <td>...</td>
      <td>7.0</td>
      <td>9.0</td>
      <td>19.0</td>
      <td>33.0</td>
      <td>20.0</td>
      <td>22.0</td>
      <td>18.0</td>
      <td>16.0</td>
      <td>11.0</td>
      <td>11.0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>11.0</td>
      <td>9.0</td>
      <td>1.0</td>
      <td>NaN</td>
      <td>3.0</td>
      <td>1.0</td>
      <td>2.0</td>
      <td>2.0</td>
      <td>1.0</td>
      <td>5.0</td>
      <td>...</td>
      <td>14.0</td>
      <td>18.0</td>
      <td>35.0</td>
      <td>41.0</td>
      <td>38.0</td>
      <td>24.0</td>
      <td>22.0</td>
      <td>37.0</td>
      <td>21.0</td>
      <td>11.0</td>
    </tr>
    <tr>
      <th>0</th>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>...</td>
      <td>2.0</td>
      <td>2.0</td>
      <td>4.0</td>
      <td>4.0</td>
      <td>1.0</td>
      <td>2.0</td>
      <td>2.0</td>
      <td>3.0</td>
      <td>1.0</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
<p>10 rows × 48 columns</p>
</div>


 
역시 Result DataFrame을 만들고 거기에 채워 넣는 방식으로 실행하였습니다.<br>
Index는 고도로서 *100 ft를 해주면 되겠습니다. 1이면 100ft 인 셈이지요.<br>
이를 일변화 시계열로 나타내면 동계와 하계는 아래와 같습니다. 







<a></a> 




![png]({{ '/assets/' | prepend: site.url }}/rjcc_metar_analysis_files/rjcc_metar_analysis_46_1.png)






![png]({{ '/assets/' | prepend: site.url }}/rjcc_metar_analysis_files/rjcc_metar_analysis_47_0.png)

 
동계 저운고 분포를 보면 강설로 인한 저운고는 비중이 낮음을 볼 수 있습니다.
이 말은 눈이 오더라도 저운고를 동반하는 경우는 상대적으로 낮은편이라 볼 수 있겠습니다.

하계의 경우 앞서 저시정 기상현상군 분포에서도 보셨다시피, 13:30Z에 비중이 높음을 알수 있습니다.<br>이 말은 안개로 인한 저운고가
높다는 것 입니다.

METAR데이터를 가지고 차트를 그리면서 기본적인 분석을 하였습니다.<br>

외에도 전세계 공항에 대한 METAR를 수집하고 분석하고 있습니다. 세계 여러 공항의 METAR를 분석해 보면서 깨닫게 된것은, 나라마다 그들의 문화가 METAR에도 반영된고 있다라는 것입니다. 표준이 있음에도 다양한 문화에서 나오는 미묘한 부분들이 반영되고 있는 것이지요.
<br>이런 미묘한 차이때문에 저는 다시 정규식을 짜야 하겠지만요.

그럼 다음 포스트에서는 과거 태풍데이터와 METAR데이터를 가지고 머신러닝을 적용한 사례로 이어가 볼까 합니다.
