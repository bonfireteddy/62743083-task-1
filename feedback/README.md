# 62743083-task-1 Map Matching


## 📦 사용 라이브러리

| 목적          | 라이브러리                  | 비고                                                |
| ----------- | ---------------------- | ------------------------------------------------- |
| Jupyter 실행  | **Anaconda**             | 파이썬 환경 및 데스크탑 실행 관리 
| 인터페이스  | **Jupyter Notebook**             | 셀 기반 실험 및 시각화
| CSV·데이터 처리  | **pandas**             | `read_csv`, `apply`, 정렬·저장 등                      |
| OSM(XML) 파싱 | **xmltodict**          | 한 줄로 XML → 딕셔너리 변환                                |
| 지오메트리 엔진    | **shapely**            | `Point`, `LineString`, `nearest_points`, 거리·투영 계산 |
| 파일 순회       | **glob**, **pathlib.Path** | 여러 GPS CSV 일괄 처리                                  |

```bash
pip install pandas xmltodict shapely
```

<br>

## 🗺️ 데이터 구조

| 이름          | 형태                    | 설명                                           |
| ----------- | --------------------- | -------------------------------------------- |
| nodes     | 노드ID: (lon, lat)  | <node ..> 태그 모두를 딕셔너리로 저장                  |
| all_roads | {wayID: LineString} | <way> → 노드ID → 좌표 리스트 → LineString(노드≥2) |
| df      | DataFrame             | GPS CSV + 매칭 컬럼 포함                           |

<br>

## 🔑 핵심 함수

### 1) OSM 파싱 → **nodes**, **all_roads**

```python
# nodes
a = osm['osm']['node']
if isinstance(a, dict):
    a = [a]
nodes = {int(n['@id']): (float(n['@lon']), float(n['@lat'])) for n in a}

# ways
ways = osm['osm']['way']
if isinstance(ways, dict): ways = [ways]
all_roads = {}
for w in ways:
    coords = [nodes[nid] for nid in map(int, (nd['@ref'] for nd in w['nd'])) if nid in nodes]
    if len(coords) >= 2:
        all_roads[int(w['@id'])] = LineString(coords)
```

* 누락 노드가 있으면 자동으로 제외 → 길이만 조금 짧아짐.

### 2) GPS 한 점 → 가장 가까운 도로 찾기

```python
DEG2M = 111_320  # 위도 1° ≈ 111.32 km (서울 기준 근사)

def find_closest_road(lon, lat, roads):
    p = Point(lon, lat)  # (경도, 위도) 순서!
    best_way, best_d, best_proj = None, float('inf'), None
    for wid, line in roads.items():
        d = p.distance(line)  # 단위: 도(degree)
        if d < best_d:
            best_way, best_d = wid, d
            best_proj = nearest_points(p, line)[1]  # 수직 투영점
    return best_way, best_d * DEG2M, best_proj  # metres
```

### 3) match_row() – DataFrame apply용

```python
def match_row(row):
    wid, dist_m, proj = find_closest_road(row.Longitude, row.Latitude, all_roads)
    return pd.Series({
        'matched_way':  wid,
        'match_dist_m': dist_m,
        'proj_lat':     proj.y,  # 위도
        'proj_lon':     proj.x   # 경도
    })

df[['matched_way','match_dist_m','proj_lat','proj_lon']] = df.apply(match_row, axis=1)
df['matched_way'] = df['matched_way'].astype('Int64')
```

| 최종 컬럼                  | 의미            |
| ---------------------- | ------------- |
| `matched_way`          | 가장 가까운 도로(ID) |
| `match_dist_m`         | 도로까지 수직 거리(m) |
| `proj_lat`, `proj_lon` | 도로 위 투영 좌표    |

<br>

## 🧹 오차(노이즈) 필터

```python
df['noisy'] = (df['match_dist_m'] > 30) | (df['HDOP'] >= 3)
clean_df = df[~df['noisy']].reset_index(drop=True)
```

* **거리 30 m 초과** 또는 **HDOP ≥ 3** → `noisy=True`.


<br>

## 🔄 GPS CSV 일괄 처리

```python
for csv in glob('../gps_logs/gps_*.csv'):
    df = pd.read_csv(csv)
    df[['matched_way','match_dist_m','proj_lat','proj_lon']] = df.apply(match_row, axis=1)
    df.to_csv(csv.replace('.csv', '_matched.csv'), index=False)
```

* 기존 파일 이름 + `_matched.csv` 로 결과 저장.

<br>

## ✅ 현재까지 완료된 내용

1. **OSM 파싱 → LineString** 구축
2. **GPS → 도로 매칭**(거리·투영 좌표 계산)
3. **노이즈 판별**(거리·HDOP)
4. 10개 GPS 로그 모두 `*_matched.csv` 생성 완료




