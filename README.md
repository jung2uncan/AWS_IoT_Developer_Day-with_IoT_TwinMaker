# AWS_IoT_Developer_Day-with_IoT_TwinMaker
IoT SiteWise와 TwinMaker를 활용한 Factory Monitoring 시스템 구축하기(With Grafana)


## 📖 Goal
- AWS IoT TwinMaker로 구축한 Digital Twin MVP 구축하기
- 장비의 IoT 데이터 수집, 어플리케이션, 데이터 분석 결과 등을 연계하는 것 포함
- 결과물은 Amazon Managed Grafana를 통해서 배포되어야 함     

  
## AWS Service used
- Cloud9
- IoT Core
- SiteWise
- TwinMaker
- Amazon Managed Service for Grafana
- Amazon Timestream

## Architecture
![Archi](https://user-images.githubusercontent.com/74287598/177700115-62cc524c-78fb-4cbb-9d1f-a05e094c8e00.PNG)

-----------------
### 1. IoT Data Generator를 이용하여 Data Source 생성하기
AWS TwinMaker를 통해서 digital twin을 만드는 경우와 같이, 실제 IoT Sensor가 아닌 데이터 시뮬레이터를 통해 동작을 테스트를 해봐야 한다. 해당 과제에서는 IoT Data Generator를 활용하여 Timestream으로 데이터를 모아 Grafana로 보여주도록 진행한다.

_참고 Git 주소 : https://github.com/kyopark2014/iot-data-generator.git_

IoT Data Generator는 Python으로 각종 IoT Data Souce를 생성하여, MQTT로 IoT Core로 보낼 수 있다.
위 Git주소의 내용을 참고하여, Data 생성/Thing으로 등록하고, MQTT test client에서 동작이 정상적인지 확인한다.
(IoT Data Generator 단계까지 설정 및 테스트를 마치고, 본 과제를 진행한다.)
<br><br>


### 2. TwinMaker Workspace 생성하기
Workspace는 digital twin 애플리케이션의 최상위 container이다다. 이 workspace내에서 digital twin 에 대한 논리적 entities, components, scene assets 및 기타 리소스 집합을 만든다.
<br>
#### Create workspace IAM Role   
Cloud9 terminal에서 아래의 명령을 수행하여, TwinMaker workspace를 위한 IAM Role을 생성
```
aws cloudformation create-stack --stack-name TwinMakerWorkspaceIAM --template-body https://sehyul-us-east-1.s3.amazonaws.com/twinmaker-workshop/twinmaker-workspace-myworkspace-account-iad.yaml --parameters ParameterKey=ResourcePrefix2,ParameterValue=워크스페이스명 ParameterKey=S3BucketPrefix,ParameterValue=myworkspace --capabilities CAPABILITY_NAMED_IAM
```
<br>

#### 생성 결과
- worksapce 명 : MyWorkspace
- S3 bucket: Create an S3 bucket
- Execution Role: 이전 단계에서 CloudFormation을 통하여 생성된 IAM Role  
![twinmaker_workspace생성](https://user-images.githubusercontent.com/74287598/177672237-3a6742a7-3d3d-441b-9dc0-e2437c9b0cf2.PNG)

<br>

### 3. IoT SiteWise Connector

IoT SiteWise connector를 활용하여 component를 생성한다. 
<br>

#### IoT Core를 활용하여 Thing 만들기
- Platform : Linux/OSX
- AWS IoT Device SDK : Python
- Thing Name : data-generator

![Thing 등록](https://user-images.githubusercontent.com/74287598/177673775-efe8587e-00ac-496c-a28d-3a08ee10c0a0.PNG)


#### Asset model 생성하기
<br>
 * Asset Model과 Asset 이란?<br>
    쉽게 말해 클래스와 인스턴스의 관계라고 볼 수 있다. Asset Model은 큰 틀을 잡아놓는 것이고, Asset은 Model을 기반으로 alias를 변경하여 각기 다른 Device에서 데이터를 수집할 수 있도록 한다.
 <br>
 __Ex) 공장 온도/습도 등을 측정하는 Asset Model__
 - 위 Asset Model을 기반으로 각 Asset 구성. 수집할 경로 alias를 다르게 주어 생성한다. 
   1번 공장 (plant/1/temperature) / 2번 공장(plant/2/temperature) / 3번 공장(plant/3/temperature) 

<br>

#### 본 과제 Entity & Component Structure
<p align="center"><img src="https://user-images.githubusercontent.com/74287598/177675085-3a4fc633-9cc2-437d-9822-67626c092ac5.PNG" height= "500" width="500"></p>

#### Asset Model
![Asset Model](https://user-images.githubusercontent.com/74287598/177674858-cba4614b-d6bd-4bd4-b129-6dbf7b123e1f.PNG)

#### Asset Model Hierarchy
![Asset Model Hierarchy](https://user-images.githubusercontent.com/74287598/177674906-081bfa46-fad7-4181-8969-0c3e85fac228.PNG)

#### 생성 예) Asset Model Detail : Ambient
![Asset Model Detail_Ambinet](https://user-images.githubusercontent.com/74287598/177677333-9428adce-4047-46cb-a947-2e5e60b0f3ef.PNG)


<br>
#### Asset 생성하기
![Asset Create](https://user-images.githubusercontent.com/74287598/177700246-47946789-5703-490f-bfce-9cf879a4d813.PNG)


#### 생성 예) Asset Detail : Ambient

![Asset Create3_Ambient Humidity](https://user-images.githubusercontent.com/74287598/177700172-2f6d937a-6297-467f-9029-5c3f9799624c.PNG)

- Model: Ambient
- Name: Ambient Asset1
  
![Asset Create1_Ambient](https://user-images.githubusercontent.com/74287598/177700126-19e0a43c-bed0-4037-ae46-0d1adecc624e.PNG)
![Asset Create2_Ambient](https://user-images.githubusercontent.com/74287598/177700128-e35d2308-c748-411e-8858-4049ce600e27.PNG)


#### IoT Core Rule 생성하기
IoT Core를 활용하려 수집된 메세지를 IoT SiteWise로 전달하는 Rule 생성과정
![IoT Core_Rule](https://user-images.githubusercontent.com/74287598/177691215-92ca97ea-a333-4019-b3b1-765f2b589b6a.PNG)

##### IoT Core > Message Routing > Rules : Ambient
SQL문을 통해 'sim/test' Topic에 대한 Message 수집한다.
![IoT Core_Rule_Ambinet](https://user-images.githubusercontent.com/74287598/177691380-f17e547c-d7fb-4eaf-9ab7-a29a98321a64.PNG)

##### IoT Core > Message Routing > Rules Action 작성 시 유의사항: Ambient
Entry에 대한 Row 작성시 변수값은 각각 고유해야한다. (수집 Device의 특징이 들어간 변수로 설정하면 좋음)
![IoT Core_Rule Action](https://user-images.githubusercontent.com/74287598/177691278-ae1e29ce-82ba-4574-92fa-efe5ff515cf4.PNG)

IoT Core서비스가 IoT SiteWise와 같은 다른 서비스에 접근하기 위한 permmision을 부여하기 위하여 IAM Role을 선택해야하는데,  새로운 rule을 생성하기 위하여 Create new role을 생성하여 적용한다.

##### Role Name : iot_rule_sitewise_role
![IoT Core_Rule 생성시 IAM Role 생성](https://user-images.githubusercontent.com/74287598/177691886-cd330fbd-86ac-4505-a8b3-c94cdbb61af5.PNG)

#### IoT SiteWise로 메세지 전송
Cloud9을 통해 메세지를 전송하고, SiteWise에서 정상 수신이 되는지 확인한다.
###### AWS IoT SiteWise > Assets > Ambinet Asset1 > Latest Value 확인
![Asset Measurement_Ambinet](https://user-images.githubusercontent.com/74287598/177692488-3feb6349-e08f-4b52-801f-889eab439983.PNG)


이 때, IoT Core의 Policy는 각 Topic을 수신할 수 있도록 설정되어야 한다. 이번 과제에서는 모든 메세지를 수신하도록 '*' 처리하였지만, 필요에 따라 Topic을 추가하면 된다.
  
![IoT Core Policies](https://user-images.githubusercontent.com/74287598/177692204-2bbfa456-6efc-445f-bfef-17bc11536434.PNG)

#### TwinMaker에서의 Create Entity/Component
- Create entity : Factory
- Create child entity : Motor, Tank, Ambient, Battery
![twinmaker_workspace생성](https://user-images.githubusercontent.com/74287598/177692851-692d03ed-cf64-4191-b6a9-3b7050671726.PNG)

#### Entities
![TwinMaker_Entities](https://user-images.githubusercontent.com/74287598/177693329-b7f5b22f-2a01-4478-b3fe-f34742be94f6.PNG)

#### Entity Detail_Component Setting : Ambient
- Component name : ambient_component
- Type: com.amazon.iotsitewise.connector
- Component type : Sitewise
- Asset model : Ambient
- Asset : Ambinet Asset1
![TwinMaker_Entity_Ambient](https://user-images.githubusercontent.com/74287598/177693484-b9eeb259-52f9-47fb-ba68-cea235d031ef.PNG)

### TwinMaker Scene 구성하기
AWS IoT TwinMaker > Workspaces > MyWorkspace > Resource Library 으로 이동 후에, Resource Library를 클릭하고 Add resources를 클릭하여 .glb(3D 이미지) Upload  (* _이미지 라이선스 확인 필수_)
![TwinMaker Scene _Resource lib](https://user-images.githubusercontent.com/74287598/177694312-2f6b9192-0cff-4a2c-a819-818d5176c055.PNG)


AWS IoT TwinMaker > Workspaces > MyWorkspace > Scenes > Create Scenes
Resource Library에 업로드한 이미지로 원하는 도면을 그린다.
![twinmaker_3D Scene](https://user-images.githubusercontent.com/74287598/177696611-861a131e-ada9-4174-a23a-e6a81fc58604.PNG)
![twinmaker_3D Scene2](https://user-images.githubusercontent.com/74287598/177696793-435c81b6-5bd5-488f-97de-4bf86e719ebd.PNG)
![twinmaker_3D Scene3](https://user-images.githubusercontent.com/74287598/177696806-8f7e1c1f-ef0f-41f1-8a3e-dbcf961231ed.PNG)

### Grafana 대시보드 통합
Grafna 연동 방법은 생략.
![Grafana Dashboards](https://user-images.githubusercontent.com/74287598/177698920-e5d6008d-2dc0-4151-9026-d5ed90c2425e.png)

