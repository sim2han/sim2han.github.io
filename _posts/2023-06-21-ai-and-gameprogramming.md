---
layout: post
title: "렌더링 과정 모사와 다양한 upscaling"
description: "인공지능과게임프로그래밍 프로젝트"
modified: 2025-11-11
tags: [c++, rendering]
categories: [project]
---

![main](/images/posts/ai-game/main.png)

### 요약

KhuGle 프레임워크 위에서 렌더링의 레스터라이징, 셰이딩 과정을 C++로 구현한다.

거기에 checkerboard, nearest-neighbor, bilinear, bicubic interpolation을 적용해보았다.

## 서론

인공지능과게임프로그래밍 수업은 충돌 검출, 렌더링, jpeg 압축, MLP, RANSAC Image Stitching등 다양한 주제를 다루었다.

여기서 과제로 한 주제를 골라 확장해보기가 제시되어 렌더링 쪽을 더 제대로 구현해보기로 했다.

구현 뒤에는 CPU로 계산하다보니 결과가 만족스럽지 않아, 다양한 업스케일링 방법을 써보며 성능을 향상하는걸 목표로 했다.

## 구현

### ObjLoader
````
class ObjLoader 
{ 
public: 
 std::vector<CKgVector3D> vertexs; 
 std::vector<std::tuple<int, int, int>> faces; 
 std::vector<std::tuple<int, int, int>> faceNorms; 
 std::vector<std::tuple<int, int, int>> faceUvs; 
 std::vector<CKgVector3D> normals; 
 std::vector<CKgVector3D> uvs; 
public: 
void LoadObjFromFile(std::string filepath, bool inverse = false) { 
 int z = inverse ? -1 : 1; 
 std::ifstream fin; 
 fin.open(filepath.c_str()); 
 
 std::string line; 
 while (std::getline(fin, line)) { 
  if (line.length() != 0) { 
   if (line[0] == 'v' && line[1] == ' ') { 
    std::stringstream ss(line); 
    std::string var, one, two, three; 
    ss >> var >> one >> two >> three; 
    vertexs.push_back(CKgVector3D(std::stold(one), std::stold(two), std::stold(three) * z)); 
   } 
   else if (line[0] == 'f' && line[1] == ' ') { 
 
    std::stringstream ss(line); 
    std::string var, v0, v0n, v0t, v1, v1n, v1t, v2, v2n, v2t, temp; 
    getline(ss, temp, ' '); // f 
 
    getline(ss, temp, ' '); 
    std::stringstream sss(temp); 
    getline(sss, v0, '/'); 
    getline(sss, v0t, '/'); 
    getline(sss, v0n); 
 
    getline(ss, temp, ' '); 
    std::stringstream ssss(temp); 
    getline(ssss, v1, '/'); 
    getline(ssss, v1t, '/'); 
    getline(ssss, v1n); 
 
    getline(ss, temp); 
    std::stringstream sssss(temp); 
    getline(sssss, v2, '/'); 
    getline(sssss, v2t, '/'); 
    getline(sssss, v2n); 
 
    faces.push_back(std::make_tuple(std::stoi(v0) - 1, std::stoi(v1) - 1, std::stoi(v2) - 1)); 
    faceNorms.push_back(std::make_tuple(std::stoi(v0n) - 1, std::stoi(v1n) - 1, std::stoi(v2n) - 1)); 
    faceUvs.push_back(std::make_tuple(std::stoi(v0t) - 1, std::stoi(v1t) - 1, std::stoi(v2t) - 1)); 
   } 
   else if (line[0] == 'v' && line[1] == 'n' && line[2] == ' ') { 
    std::stringstream ss(line); 
    std::string var, one, two, three; 
    ss >> var >> one >> two >> three; 
    normals.push_back(CKgVector3D(std::stod(one), std::stod(two), std::stod(three) * z)); 
   } 
   else if (line[0] == 'v' && line[1] == 't' && line[2] == ' ') { 
    std::stringstream ss(line); 
    std::string var, one, two; 
    ss >> var >> one >> two; 
    uvs.push_back(CKgVector3D(std::stod(one), std::stod(two), 0)); 
   } 
  } 
 } 
 std::cout << "ModelInfo Vertex:" << vertexs.size() << " Faces:" << faces.size() << std::endl; 
} 
};
````

obj파일을 사용한다. obj포맷은 위치가 v 1 2 3, 노말이 vn 1 2 3, uv가 vt 1 2, 이런 식으로 한 줄로 표현되고, face는 1/1/1 2/2/2 3/3/3 이렇게 세 점의 위치, 노말, uv를 지정해 정의한다.

문자열로 되어있는 버텍스 정보를 읽어 배열에 저장하고, face는 해당하는 점을 찾아 연결하면 된다.

최종적으로 점의 위치, 노말, uv가 각각의 벡터에 들어가고, face가 가지는 점의 정보가 인덱스로서 저장된다.

프로그램과 좌표계에 따라 z축이 뒤집히는 경우가 있어 인자로 inverse를 추가해 z축을 뒤집어 읽을지 결정한다.

### rendering

````
// Draw Mesh 
DrawTriangleMesh(Parent->m_ImageR, Parent->m_ImageG, Parent->m_ImageB, 
  Parent->m_nW, Parent->m_nH, 
  Projected, VertexNormal, VertexUv, texture, texture2, ZBuffer, m_fgColor); 
````

수업에서 제시된 DrawTriangle()은 삼각형의 엣지만 그리는 함수였다.

DrawTriangleMesh()를 새로 정의해 앞서 읽은 삼각형 정보로 Mesh를 그릴 수 있게 만든다.

````
void CKhuGle3DSprite::DrawTriangleMesh(unsigned char** R, unsigned char** G, unsigned char** B, int nW, int nH, 
 CKgTriangle& tri, CKgTriangle& normal, CKgTriangle& uv, CKhuGleSignal& texture, CKhuGleSignal& texture2, double** ZBuffer, KgColor24 
Color24) 
{ 
 // 레스터화 
 
 // 범위 지정 
 int recStartX = std::clamp(min(min(tri.v0.x, tri.v1.x), tri.v2.x) - 1, 0., 10000.); 
 int recStartY = std::clamp(min(min(tri.v0.y, tri.v1.y), tri.v2.y) - 1, 0., 10000.); 
 int recEndX = std::clamp(max(max(tri.v0.x, tri.v1.x), tri.v2.x) + 1, 0., 10000.); 
 int recEndY = std::clamp(max(max(tri.v0.y, tri.v1.y), tri.v2.y) + 1, 0., 10000.); 
 
 for (int x = recStartX; x < recEndX; x++) 
  for (int y = recStartY; y < recEndY; y++) 
  { 
   // ..... 여러 처리들 
 
   R[y][x] = KgGetRed(color); 
   G[y][x] = KgGetGreen(color); 
   B[y][x] = KgGetBlue(color); 
 
   renderPixel++; 
  } 
}
````

함수는 먼저 입력 받은 삼각형을 감싸는 사각형 범위를 정한다.

이제 사각형 내부의 픽셀을 순서대로 돌아다니며 여러 계산을 할 것이다. 

#### 레스터화

````
 CKgVector3D point(x, y, 0); 
 
 CKgVector3D v0(tri.v0.x, tri.v0.y, 0); 
 CKgVector3D v1(tri.v1.x, tri.v1.y, 0); 
 CKgVector3D v2(tri.v2.x, tri.v2.y, 0); 
 
 CKgVector3D v0tov1 = v1 - v0; 
 CKgVector3D v0tov2 = v2 - v0; 
 CKgVector3D v0toP = point - v0; 
 CKgVector3D v1tov2 = v2 - v1; 
 CKgVector3D v2tov0 = v0 - v2; 
 
 // 삼각형 내부의 점 찾기 
 if (!(v0tov1.x * (point - v0).y - v0tov1.y * (point - v0).x <= 0 && v1tov2.x * (point - v1).y - v1tov2.y * (point - v1).x <= 0))  
   v2tov0.x * (point - v2).y - v2tov0.y * 
````

해당 점이 삼각형 내부의 점인지 확인한다. 삼각형의 한 점에서 한 변의 벡터와 점까지의 벡터를 외적하여 z값을 확인하면 점이 변을 기준으로 어느 방향에 있는지 알 수 있다. 이를 통해 점이 모든 변에서 안쪽에 존재하면 점은 삼각형이 그릴 수 있는 점임을 확인할 수 있다. 

#### 버텍스 보간

````
// 버텍스 값 보간 
double b = (v0toP.y * v0tov2.x - v0tov2.y * v0toP.x) / (v0tov2.x * v0tov1.y - v0tov2.y * v0tov1.x); 
double a = (v0toP.x - v0tov1.x * b) / v0tov2.x; 
double v0value = 1 - a - b; 
double v1value = b; 
double v2value = a; 
````

점의 위치를 v0에서 삼각형 꼭지점 v1과 v2의 벡터로 나타낼 수 있다. 연립 일차방정식을 통해 가중치 a와 b를 계산하고, 1-a-b, a, b를 각 버텍스의 가중치로 설정한다.

![vertex-interpolation](/images/posts/ai-game/vertex-interpolation.png)

#### 깊이 테스트

````
// 깊이 값 테스트 
double z = (tri.v0.z * v0value + tri.v1.z * v1value + tri.v2.z * v2value); 
if (ZBuffer[y][x] <= z) 
    continue; 
else if (z <= 0) 
    continue; 
ZBuffer[y][x] = z; 
````

버텍스 변환 시 z값은 실제 화면에서의 깊이를 나타낸다. 버텍스의 z값을 보간해 픽셀의 z값을 얻을 수 있고, 해당 정보를 이미 그려진 픽셀과 비교해 앞에 있을 때만 픽셀을 그린다. 또, 픽셀이 카메라 안으로 들어간 경우도 그리지 않는다. 

#### 셰이딩

````
// 여기서부터 셰이더 부분 
CKgVector3D normalvec = v0value * normal.v0 + v1value * normal.v1 + v2value * normal.v2; 
CKgVector3D uvVec = v0value * uv.v0 + v1value * uv.v1 + v2value * uv.v2; 

double dot = normalvec.Dot(CKgVector3D(1, 1, 1)); 
dot = min(max(dot, 0), 1.0); 

KgColor24 color = 0xFFFFFF; 

if (dot > 0.5) 
    color = texture.ReadTexture(uvVec.x, uvVec.y); 
else 
    color = texture2.ReadTexture(uvVec.x, uvVec.y); 
R[y][x] = KgGetRed(color); 
G[y][x] = KgGetGreen(color); 
B[y][x] = KgGetBlue(color); 
````

버텍스로부터 uv와 노말을 제대로 보간 했다면 셰이딩은 간단하다. 임의의 빛 방향 (1, 1, 1)과 노말을 내적해 픽셀의 밝기를 계산한다. 그리고 셀 셰이딩을 위해, 내적 값을 0.5를 기준으로 나눠 밝은 부분과 어두운 부분을 나누었다. 각 부분에 따라 밝은 텍스처와 어두운 텍스처를 읽고 색을 버퍼에 기록한다. 

![shading](/images/posts/ai-game/shading.png)

### interpolation

#### checkerboard

````
// x2 체커보드 렌더링 
if (X2CHECKORBOARD == true) 
 if (!(((x + 1) % 4 / 2 == 0) != ((y + 1) % 4 / 2 == 0))) 
  continue; 
 
if (X2CHECKORBOARD) { 
 for (int y = 0; y < Parent->m_nH / 4; y++) 
  for (int x = 0; x < Parent->m_nW / 4; x++) { 
   x *= 4; y *= 4; 
   if (x % 4 == 0 && y % 4 == 0) { 
    Parent->m_ImageR[y][x] = (Parent->m_ImageR[y + 1][x] + Parent->m_ImageR[y][x + 1]) / 2; 
    Parent->m_ImageR[y + 1][x + 1] = Parent->m_ImageR[y][x]; 
    Parent->m_ImageG[y][x] = (Parent->m_ImageG[y + 1][x] + Parent->m_ImageG[y][x + 1]) / 2; 
    Parent->m_ImageG[y + 1][x + 1] = Parent->m_ImageG[y][x]; 
    Parent->m_ImageB[y][x] = (Parent->m_ImageB[y + 1][x] + Parent->m_ImageB[y][x + 1]) / 2; 
    Parent->m_ImageB[y + 1][x + 1] = Parent->m_ImageB[y][x]; 
   } 
   y += 3; 
   if (x % 4 == 0 && y % 4 == 3) { 
    Parent->m_ImageR[y][x] = (Parent->m_ImageR[y - 1][x] + Parent->m_ImageR[y][x + 1]) / 2; 
    Parent->m_ImageR[y - 1][x + 1] = Parent->m_ImageR[y][x]; 
    Parent->m_ImageG[y][x] = (Parent->m_ImageG[y - 1][x] + Parent->m_ImageG[y][x + 1]) / 2; 
    Parent->m_ImageG[y - 1][x + 1] = Parent->m_ImageG[y][x]; 
    Parent->m_ImageB[y][x] = (Parent->m_ImageB[y - 1][x] + Parent->m_ImageB[y][x + 1]) / 2; 
    Parent->m_ImageB[y - 1][x + 1] = Parent->m_ImageB[y][x]; 
   } 
   y -= 3; x += 3; 
   if (x % 4 == 3 && y % 4 == 0) { 
    Parent->m_ImageR[y][x] = (Parent->m_ImageR[y + 1][x] + Parent->m_ImageR[y][x - 1]) / 2; 
    Parent->m_ImageR[y + 1][x - 1] = Parent->m_ImageR[y][x]; 
    Parent->m_ImageG[y][x] = (Parent->m_ImageG[y + 1][x] + Parent->m_ImageG[y][x - 1]) / 2; 
    Parent->m_ImageG[y + 1][x - 1] = Parent->m_ImageG[y][x]; 
    Parent->m_ImageB[y][x] = (Parent->m_ImageB[y + 1][x] + Parent->m_ImageB[y][x - 1]) / 2; 
    Parent->m_ImageB[y + 1][x - 1] = Parent->m_ImageB[y][x]; 
   } 
   y += 3; 
   if (x % 4 == 3 && y % 4 == 3) { 
    Parent->m_ImageR[y][x] = (Parent->m_ImageR[y - 1][x] + Parent->m_ImageR[y][x - 1]) / 2; 
    Parent->m_ImageR[y - 1][x - 1] = Parent->m_ImageR[y][x]; 
    Parent->m_ImageG[y][x] = (Parent->m_ImageG[y - 1][x] + Parent->m_ImageG[y][x - 1]) / 2; 
    Parent->m_ImageG[y - 1][x - 1] = Parent->m_ImageG[y][x]; 
    Parent->m_ImageB[y][x] = (Parent->m_ImageB[y - 1][x] + Parent->m_ImageB[y][x - 1]) / 2; 
    Parent->m_ImageB[y - 1][x - 1] = Parent->m_ImageB[y][x]; 
   } 
   x -= 3; y -= 3; x /= 4; y /= 4; 
  } 
 } 
````

렌더링을 2x2 크기 체크보드 모양으로 한다. 빈 칸은 각각 주변에 2개의 렌더링 된 픽셀을 가지므로 해당 값의 평균으로 보간 한다. 코드는 4x4로 칸을 나눠 네 모퉁이에 대해 각각 보간 하는 식으로 진행했다. 

![checkerboard](/images/posts/ai-game/checkerboard.png)

#### nearest-neighbor

````
if (X4PIXELSCALE) 
 if (x % 4 != 0 || y % 4 != 0) 
  continue; 
 
if (X4PIXELSCALE) { 
 for (int y = 0; y < Parent->m_nH / 4 - 1; y++) 
  for (int x = 0; x < Parent->m_nW / 4 - 1; x++) { 
   x *= 4; y *= 4; 
   for (int yy = 0; yy < 4; yy++) 
    for (int xx = 0; xx < 4; xx++) { 
     if (yy == 0 && xx == 0) 
      continue; 
     Parent->m_ImageR[y + yy][x + xx] = Parent->m_ImageR[y][x]; 
     Parent->m_ImageG[y + yy][x + xx] = Parent->m_ImageG[y][x]; 
     Parent->m_ImageB[y + yy][x + xx] = Parent->m_ImageB[y][x]; 
      } 
   x /= 4; y /= 4; 
  } 
 } 
````

Nearest-Neighbor은 해당 픽셀에서 가장 가까운 픽셀을 찾아 렌더링한다. 여기서는 편의를 위해 4*4의 사각형으로 나누고 좌상단의 픽셀을 기준으로 색을 결정하도록 했다.

![nearest](/images/posts/ai-game/nearest.png)

#### bilinear

````
if (X4BLINEAR) 
 if (x % 4 != 0 || y % 4 != 0) 
  continue; 
if (X4BLINEAR) { 
 for (int y = 1; y < Parent->m_nH / 4 - 1; y++) 
  for (int x = 1; x < Parent->m_nW / 4 - 1; x++) { 
   x *= 4; y *= 4; 
   for (int yy = 0; yy < 4; yy++) 
    for (int xx = 0; xx < 4; xx++) { 
     if (yy == 0 && xx == 0) 
      continue; 
 Parent->m_ImageR[y + yy][x + xx] = ((Parent->m_ImageR[y][x + 4] * xx + Parent->m_ImageR[y][x] * (4 - xx)) * (4 - yy) + (Parent->m_ImageR[y + 4][x + 4] * xx + 
Parent->m_ImageR[y + 4][x] * (4 - xx)) * (yy)) / 16; 
 Parent->m_ImageG[y + yy][x + xx] = ((Parent->m_ImageG[y][x + 4] * xx + Parent->m_ImageG[y][x] * (4 - xx)) * (4 - yy) + (Parent->m_ImageG[y + 4][x + 4] * xx + 
Parent->m_ImageG[y + 4][x] * (4 - xx)) * (yy)) / 16; 
 Parent->m_ImageB[y + yy][x + xx] = ((Parent->m_ImageB[y][x + 4] * xx + Parent->m_ImageB[y][x] * (4 - xx)) * (4 - yy) + (Parent->m_ImageB[y + 4][x + 4] * xx + 
Parent->m_ImageB[y + 4][x] * (4 - xx)) * (yy)) / 16; 
   } 
   x /= 4; y /= 4; 
  } 
 } 
````

x2bilinear와 원리가 같지만 4*4 크기에서 실행한다. 자신의 좌표에 따라 주변 4개 픽셀의 가중치를 계산한다. 먼저 4개의 픽셀에 대해 x축을 기준으로 2개의 보간을 실시하고, 두 값을 다시 y축으로 보간 한다.

![bilinear](/images/posts/ai-game/bilinear.png)

#### bicubic

<details>
<summary>코드 보기</summary>

````
if (X4CUBIC) 
 if (x % 4 != 0 || y % 4 != 0) 
  continue; 
 
if (X4CUBIC) { 
 for (int y = 1; y < Parent->m_nH / 4 - 2; y++) 
  for (int x = 1; x < Parent->m_nW / 4 - 2; x++) { 
   x *= 4; y *= 4; 
   for (int yy = 0; yy < 4; yy++) 
    for (int xx = 0; xx < 4; xx++) { 
     if (yy == 0 && xx == 0) 
      continue; 
 
    double r1 = (3 * (4 + xx) * Parent->m_ImageR[y - 4][x] * (4 - xx) * (8 - xx) 
     + 3 * (4 + xx) * xx * Parent->m_ImageR[y - 4][x + 4] * (8 - xx) 
     - Parent->m_ImageR[y - 4][x - 4] * xx * (4 - xx) * (8 - xx) 
     - Parent->m_ImageR[y - 4][x + 8] * xx * (4 - xx) * (4 + xx)) / (6 * 4 * 4 * 4); 
    double r2 = (3 * (4 + xx) * Parent->m_ImageR[y][x] * (4 - xx) * (8 - xx) 
     + 3 * (4 + xx) * xx * Parent->m_ImageR[y][x + 4] * (8 - xx) 
     - Parent->m_ImageR[y][x - 4] * xx * (4 - xx) * (8 - xx) 
     - Parent->m_ImageR[y][x + 8] * xx * (4 - xx) * (4 + xx)) / (6 * 4 * 4 * 4); 
    double r3 = (3 * (4 + xx) * Parent->m_ImageR[y + 4][x] * (4 - xx) * (8 - xx) 
     + 3 * (4 + xx) * xx * Parent->m_ImageR[y + 4][x + 4] * (8 - xx) 
     - Parent->m_ImageR[y + 4][x - 4] * xx * (4 - xx) * (8 - xx) 
     - Parent->m_ImageR[y + 4][x + 8] * xx * (4 - xx) * (4 + xx)) / (6 * 4 * 4 * 4); 
    double r4 = (3 * (4 + xx) * Parent->m_ImageR[y + 8][x] * (4 - xx) * (8 - xx) 
     + 3 * (4 + xx) * xx * Parent->m_ImageR[y + 8][x + 4] * (8 - xx) 
     - Parent->m_ImageR[y + 8][x - 4] * xx * (4 - xx) * (8 - xx) 
     - Parent->m_ImageR[y + 8][x + 8] * xx * (4 - xx) * (4 + xx)) / (6 * 4 * 4 * 4); 
    double r = (3 * (4 + yy) * r2 * (4 - yy) * (8 - yy) 
     + 3 * (4 + yy) * yy * r3 * (8 - yy) 
     - r1 * yy * (4 - yy) * (8 - yy) 
     - r3 * yy * (4 - yy) * (4 + yy)) / (6 * 4 * 4 * 4); 
    Parent->m_ImageR[y + yy][x + xx] = std::clamp(r, 0., 255.); 
 
    double g1 = (3 * (4 + xx) * Parent->m_ImageG[y - 4][x] * (4 - xx) * (8 - xx) 
     + 3 * (4 + xx) * xx * Parent->m_ImageG[y - 4][x + 4] * (8 - xx) 
     - Parent->m_ImageG[y - 4][x - 4] * xx * (4 - xx) * (8 - xx) 
     - Parent->m_ImageG[y - 4][x + 8] * xx * (4 - xx) * (4 + xx)) / (6 * 4 * 4 * 4); 
    double g2 = (3 * (4 + xx) * Parent->m_ImageG[y][x] * (4 - xx) * (8 - xx) 
     + 3 * (4 + xx) * xx * Parent->m_ImageG[y][x + 4] * (8 - xx) 
     - Parent->m_ImageG[y][x - 4] * xx * (4 - xx) * (8 - xx) 
     - Parent->m_ImageG[y][x + 8] * xx * (4 - xx) * (4 + xx)) / (6 * 4 * 4 * 4); 
    double g3 = (3 * (4 + xx) * Parent->m_ImageG[y + 4][x] * (4 - xx) * (8 - xx) 
     + 3 * (4 + xx) * xx * Parent->m_ImageG[y + 4][x + 4] * (8 - xx) 
     - Parent->m_ImageG[y + 4][x - 4] * xx * (4 - xx) * (8 - xx) 
     - Parent->m_ImageG[y + 4][x + 8] * xx * (4 - xx) * (4 + xx)) / (6 * 4 * 4 * 4); 
    double g4 = (3 * (4 + xx) * Parent->m_ImageG[y + 8][x] * (4 - xx) * (8 - xx) 
     + 3 * (4 + xx) * xx * Parent->m_ImageG[y + 8][x + 4] * (8 - xx) 
     - Parent->m_ImageG[y + 8][x - 4] * xx * (4 - xx) * (8 - xx) 
     - Parent->m_ImageG[y + 8][x + 8] * xx * (4 - xx) * (4 + xx)) / (6 * 4 * 4 * 4); 
    double g = (3 * (4 + yy) * g2 * (4 - yy) * (8 - yy) 
     + 3 * (4 + yy) * yy * g3 * (8 - yy) 
     - g1 * yy * (4 - yy) * (8 - yy) 
     - g3 * yy * (4 - yy) * (4 + yy)) / (6 * 4 * 4 * 4); 
     Parent->m_ImageG[y + yy][x + xx] = std::clamp(g, 0., 255.); 
 
    double b1 = (3 * (4 + xx) * Parent->m_ImageB[y - 4][x] * (4 - xx) * (8 - xx) 
     + 3 * (4 + xx) * xx * Parent->m_ImageB[y - 4][x + 4] * (8 - xx) 
     - Parent->m_ImageB[y - 4][x - 4] * xx * (4 - xx) * (8 - xx) 
     - Parent->m_ImageB[y - 4][x + 8] * xx * (4 - xx) * (4 + xx)) / (6 * 4 * 4 * 4); 
    double b2 = (3 * (4 + xx) * Parent->m_ImageB[y][x] * (4 - xx) * (8 - xx) 
     + 3 * (4 + xx) * xx * Parent->m_ImageB[y][x + 4] * (8 - xx) 
     - Parent->m_ImageB[y][x - 4] * xx * (4 - xx) * (8 - xx) 
     - Parent->m_ImageB[y][x + 8] * xx * (4 - xx) * (4 + xx)) / (6 * 4 * 4 * 4); 
    double b3 = (3 * (4 + xx) * Parent->m_ImageB[y + 4][x] * (4 - xx) * (8 - xx) 
     + 3 * (4 + xx) * xx * Parent->m_ImageB[y + 4][x + 4] * (8 - xx) 
     - Parent->m_ImageB[y + 4][x - 4] * xx * (4 - xx) * (8 - xx) 
     - Parent->m_ImageB[y + 4][x + 8] * xx * (4 - xx) * (4 + xx)) / (6 * 4 * 4 * 4); 
    double b4 = (3 * (4 + xx) * Parent->m_ImageB[y + 8][x] * (4 - xx) * (8 - xx) 
     + 3 * (4 + xx) * xx * Parent->m_ImageB[y + 8][x + 4] * (8 - xx) 
     - Parent->m_ImageB[y + 8][x - 4] * xx * (4 - xx) * (8 - xx) 
     - Parent->m_ImageB[y + 8][x + 8] * xx * (4 - xx) * (4 + xx)) / (6 * 4 * 4 * 4); 
    double b = (3 * (4 + yy) * b2 * (4 - yy) * (8 - yy) 
     + 3 * (4 + yy) * yy * b3 * (8 - yy) 
     - b1 * yy * (4 - yy) * (8 - yy)      - b3 * yy * (4 - yy) * (4 + yy)) / (6 * 4 * 4 * 4); 
    Parent->m_ImageB[y + yy][x + xx] = std::clamp(b, 0., 255.); 
      } 
     x /= 4; y /= 4; 
  } 
}
````

</details>

1차원에서 4개의 픽셀이 있으면 3차함수로 interpolation할 수 있다. 이것을 x, y축에 적용해 16개의 픽셀로 3차함수 그래프를 구성하고 그것으로 픽셀 값을 유추한다. 코드가 길지만 간단히, 주변 4*4픽셀에서 x축으로 보간을 4번 진행하고 해당 값으로 y축에서 보간을 한 번 더 진행하는 코드다.

## 결과

![result](/images/posts/ai-game/result.png)

각각 방법에 따른 프레임과, 실제로 렌더링한 픽셀의 개수를 표시했다.

원본은 40~60fps대를 보이고 보간 방법에 따라 따라 90fps까지도 상승한다. 하지만 실제 렌더링 하는 픽셀을 1/16까지 줄였음에도 프레임은 크게 오르지 않았는데, 그 이유는 실제 게임은 레스터화를 GPU가 최적화하기 때문에 빠른 대신 각 픽셀을 계산하는데 많은 시간이 걸리지만, 여기서는 레스터화의 시간이 오래 걸리고, 셰이딩은 간단하기 때문에 시간관계가 반대다. 따라서 렌더링하는 픽셀을 줄여도 큰 성능 향상을 볼 수 없었다.

실제 게임에서 업스케일링을 사용할 때, 텍스처를 늘리는 부분에서 Nearest, Bilinear, Bicubic을 사용하고, 렌더링에는 checkerboard 정도를 자주 사용한다. 최근에 많이 사용되는 인공지능 gpu업스케일링으로 AMD의 FSR, Nvidia의 DLSS가 사용되는데, FSR은 Lanczos방식을 포함한다. 이런 업스케일링을 통해 개발자와 사용자는 프레임과 이미지 품질 사이의 저울질을 할 수 있다. 