Monolith to Microservices with Docker and AWS Fargate
====================================================

Mythical Mysfits 팀에 오신 것을 환영합니다!

이 실습에서는 Docker를 사용하여 Monolithic Mythical Mysfits 입양 플랫폼을 구축하고 Amazon ECS에 배포한 다음 보다 관리하기 쉬운 몇 가지 마이크로 서비스로 분해합니다. 자 시작합니다!

### Requirements:

* AWS account - 계정이 없다면 무료로 쉽게 계정을 [생성](https://aws.amazon.com/)할 수 있습니다.
* 권한을 가진 AWS IAM 계정을 통해 CloudFormation, IAM, EC2, ECS, ECR, ELB / ALB, VPC, SNS, CloudWatch, Cloud9와 상호 작용할 수 있습니다. [Learn how](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html).
* [Python](https://wiki.python.org/moin/BeginnersGuide/Programmers), [Docker](https://www.docker.com/), 그리고 [AWS](httpts://aws.amazon.com) 에 대한 이해 - *필수는 아닌 보너스*.

### What you'll do:

이 실습은 순서대로 완료할 수 있게 설계되었으며 지침은 아래에 설명되어 있습니다. 실습을 완료하기 위해서는 읽고 따라하시면 됩니다. AWS 이벤트를 진행하는 경우에는 워크숍 진행자가 실습에 대한 개요를 설명하고 질문에 대한 답변을 드립니다. 진행중 문제가 생기더라도 걱정하지 않아도 됩니다. 지침 사이사이에 힌트가 제공됩니다.

* **Workshop Setup:** [AWS에서 작업 환경 설정](#lets-begin)
* **Lab 1:** [Mythical Mysfits 모놀리스 컨테이너화](#lab-1---containerize-the-mythical-mysfits-adoption-agency-platform)
* **Lab 2:** [AWS Fargate를 사용하여 컨테이너 배포](#lab-2---deploy-your-container-using-ecrecs)
* **Lab 3:** [ALB 및 ECS 서비스로 입양 플랫폼 모놀리스 확장](#lab-3---scale-the-adoption-platform-monolith-with-an-alb)
* **Lab 4:** [AWS Fargate를 사용하여 더 많은 마이크로 서비스를 점진적으로 생성하고 배포](#lab-4-incrementally-build-and-deploy-each-microservice-using-fargate)
* **Cleanup** [사용한 자원을 삭제](#workshop-cleanup)

### Conventions:

이 워크샵 전체에서는 터미널에서 실행할 명령어를 제공합니다. 다음과 같습니다:

<pre>
$ ssh -i <b><i>PRIVATE_KEY.PEM</i></b> ec2-user@<b><i>EC2_PUBLIC_DNS_NAME</i></b>
</pre>

명령은`$`다음에 시작됩니다. ***UPPER_ITALIC_BOLD*** 인 텍스트는 사용자 환경에 고유한 값을 나타냅니다. 예를 들어, ***PRIVATE\_KEY.PEM***은 계정에서 생성한 SSH 키쌍의 개인키를 나타내며 ***EC2\_PUBLIC\_DNS\_NAME***은 계정에서 시작한 EC2 인스턴스에만 해당되는 값입니다. CloudFormation 출력에서 또는 [AWS 관리 콘솔](https://console.aws.amazon.com)의 특정 서비스 대시 보드로 이동하여 이런 고유한 값을 찾을 수 있습니다.

힌트도 제공되며 다음과 같습니다:

<details>
<summary>HINT</summary>

**Nice work, you just revealed a hint!**
</details>

*힌트의 내용을 보려면 화살표를 클릭하십시오.*

### IMPORTANT: Workshop Cleanup

AWS에 인프라 배포는 비용이 발생됩니다. AWS 이벤트에 참석하는 경우 크레딧이 제공됩니다. 워크샵을 마쳤으면 [지침의 마지막 부분에 있는 단계](#workshop-cleanup)를 통해 모든 것이 삭제되었는지 확인하여 불필요한 과금이 발생되지 않도록 합니다.

## Let's Begin!

### Workshop Setup:

1. 브라우저의 새 탭에서 아래의 CloudFormation 시작 템플릿 링크를 엽니다. 링크는 CloudFormation 대시 보드를 로드하고 선택한 리전에서 스택 생성 절차를 시작합니다:
   
    아래의 **Deploy to AWS** 아이콘 중 하나를 클릭하여 워크숍 인프라를 생성하세요.

| Region | Launch Template |
| ------------ | ------------- | 
**Seoul** (ap-northeast-2) | [![Launch Mythical Mysfits Stack into Oregon with CloudFormation](/images/deploy-to-aws.png)](https://console.aws.amazon.com/cloudformation/home?region=ap-northeast-2#/stacks/new?stackName=mysfits-fargate&templateURL=https://s3.amazonaws.com/mythical-mysfits-website/fargate/core.yml)  

2. 템플릿을 통해 자동으로 CloudFormation 대시 보드로 이동하여 지정된 리전에서 스택 생성 프로세스를 시작합니다. 스택에 계정 내에서 고유한 이름을 지정하고 마법사를 통해 스택을 시작합니다. 모든 옵션을 기본값으로 유지하지만 CloudFormation에서 사용자를 대신하여 IAM 역할을 생성할 수 있도록 확인란을 선택해야합니다:

    ![IAM resources acknowledgement](images/00-cf-create.png)

    스택 실행 진행 상황은 *이벤트* 탭을 참고하십시오. 시작이 실패하면 문제에 대한 세부 정보를 볼 수도 있습니다. 스택 상태가 "CREATE_COMPLETE"로 진행되면 다음 단계로 진행합니다.

2. CloudFormation으로 생성된 AWS Cloud9 환경에 접속:

    AWS 콘솔 홈페이지에서 서비스 검색바에 **Cloud9**를 입력하고 선택합니다. "Project-***STACK_NAME***"과 같은 환경을 찾습니다:

    ![Cloud9 프로젝트 선택](images/00-cloud9-select.png)

    IDE를 열면 다음과 같은 시작 화면이 나타납니다:
    ![cloud9-welcome](images/00-cloud9-welcome.png)

    왼쪽 창(파란색)에서는 사용자 환경으로 다운로드한 모든 파일이 파일 트리에 나타납니다. 가운데(빨간색) 창에서는 열려있는 모든 문서가 표시됩니다. 왼쪽 창에서 README.md를 두 번 클릭하여 테스트하고 임의의 텍스트를 추가하여 파일을 편집합니다. 그런 다음 파일 - 저장을 클릭하여 저장합니다. 키보드 단축키도 작동합니다. 하단에는 bash shell (노란색)이 표시됩니다. 나머지 실습에서는이 셸을 사용하여 모든 명령을 입력합니다. 테마를 변경하거나 창을 이동하는 등의 방법으로 Cloud9 환경을 사용자 정의할 수도 있습니다. 어두운 테마를 좋아하는 경우 오른쪽 상단의 톱니 바퀴 아이콘을 클릭한 다음 "테마"를 클릭하고 어두운 테마를 선택하여 선택할 수 있습니다.

3. Mythical Mysfits Workshop 저장소 복제:

    새 Cloud9 IDE의 하단 패널에 터미널 명령줄을 위한 터미널이 열려 있고 사용할 준비가 되었습니다. 터미널에서 다음 git 명령을 실행하여이 학습서를 완료하는데 필요한 코드를 복제합니다:

    ```
    $ git clone https://github.com/aws-samples/amazon-ecs-mythicalmysfits-workshop.git
    ```

    저장소를 복제하면 프로젝트 탐색기에 복제된 파일이 포함된 것을 볼 수 있습니다.

    터미널에서 디렉토리를 워크샵의 하위 디렉토리로 변경합니다:

    ```
    $ cd amazon-ecs-mythicalmysfits-workshop/workshop-1
    ```

4. `setup` 스크립트를 사용하여 몇 가지 추가 자동 설정 단계를 실행합니다:

    ```
    $ script/setup
    ```

    이 스크립트는 불필요한 Docker 이미지를 삭제하여 디스크 공간을 확보하고 DynamoDB 테이블에 일부 시드 데이터를 채웁니다. 또한 사이트 자원을 S3에 업로드하고 나중에 설명할 Docker 관련 인증 메커니즘을 설치합니다. 스크립트가 완료되면 "성공!" 메시지를 확인하세요.


### Checkpoint:
이때 Mythical Mysfits 웹 사이트는 CloudFormation에서 생성한 S3 버킷의 정적 사이트 엔드 포인트에서 접근할 수 있어야합니다. <code> http://<b><i>BUCKET_NAME</i></b>.s3-website.<b><i>REGION</i></b>.amazonaws.com/</code> 사이트를 방문하십시오. 편의를 위해 콘솔의 CloudFormation 출력 탭에 링크를 만들었습니다. 또는 `workshop-1/cfn-outputs.json` 파일에 저장된 CloudFormation 출력에서 ***BUCKET_NAME***을 찾을 수 있습니다. ***REGION***은 CloudFormation 스택을 배포한 리전[code] (https://docs.aws.amazon.com/general/latest/gr/rande.html#s3_region)이어야합니다(예: 서울 지역의 경우 <i>ap-northeast-2</i>를 참조하십시오). 사이트를 볼 수 있지만 Mythical Mysfits 모놀리스 서비스를 시작할 때까지 아직 콘텐츠가 많이 표시되지 않습니다:

![initial website](images/00-website.png)

[*^ back to top*](#monolith-to-microservices-with-docker-and-aws-fargate)


## Lab 1 - Containerize the Mythical Mysfits adoption agency platform

Mythical Mysfits 입양 대행사 인프라는 항상 EC2 VM에서 직접 실행되었습니다. 첫번째 단계는 현재 Mythical Mysfits 입양 플랫폼을 컨테이너화하여 코드를 패키지화하는 방법을 현대화하는 것입니다. 이 플랫폼을 모놀리스 응용 프로그램이라고도 합니다. 이를 위해 [Dockerfile] (https://docs.docker.com/engine/reference/builder/)을 작성합니다. [Docker] (https://aws.amazon.com/의 레시피입니다) docker)를 사용하여 컨테이너 이미지를 만듭니다. [AWS Cloud9](https://aws.amazon.com/cloud9/) 개발 환경을 사용하여 Dockerfile을 작성하고 컨테이너 이미지를 빌드한 후 실행하여 입양 절차를 처리할 수 있는지 확인합니다.

[컨테이너] (https://aws.amazon.com/what-are-containers/)는 코드와 모든 종속성을 실행할 수 있도록 소프트웨어(예: 웹 서버, 프록시, 배치 프로세스 작업자)를 패키징하는 방법입니다. "잠깐, 가상 머신 (VM) 아닙니까?"라고 생각할 수도 있습니다. 컨테이너는 운영 체제를 가상화하고 VM은 하드웨어를 가상화합니다. 컨테이너는 격리, 이식성 및 반복성을 제공하므로 개발자가 불 필요하고 어려운 작업 없이 환경을 쉽게 가동하고 구축을 시작할 수 있습니다. 더 중요한 것은 컨테이너는 코드가 어디에서나 동일한 방식으로 실행되도록 보장하므로 랩톱에서 작동하면 프로덕션에서도 작동한다는 것입니다.

### Here's what you're going to work on in lab 1:

![Lab 1 Architecture](images/01-arch.png)

1. Dockerfile 초안을 검토하고 파일에 주석으로 표시된 누락된 명령을 추가합니다:

    *Note: Dockerfile이 작동하는 방식에 이미 익숙하고 모놀리스를 마이크로 서비스로 분리하는데 집중하고 싶다면 5단계 마지막 부분에 있는 ["HINT: Final Dockerfile"] (#final-dockerfile)로 건너 뜁니다. 힌트 내용이 있는 monolith 디렉토리에 Dockerfile을 만들고, "monolith" 이미지를 빌드하고 6단계를 계속합니다. 그렇지 않으면 아래를 계속하면 됩니다*

    Mythical Mysfits의 개발자중 한 명은 남는 시간에 Dockerfile 작업을 시작했지만 완료하기 전에 우선 순위가 높은 프로젝트에 불려갔습니다.

    Cloud9 파일 트리에서 `workshop-1/app/monolith-service`로 이동한 다음 **Dockerfile.draft**를 두 번 클릭하여 파일을 편집합니다.

    *Note: bash 쉘과 vi 또는 emacs와 같은 텍스트 편집기를 대신 사용하는 것도 가능합니다.*

    내용을 검토하면 파일 마지막 부분에 여전히 수행해야 할 작업에 대한 몇 가지 설명이 표시됩니다. 주석은 "#"으로 표시됩니다.

    Docker는 Dockerfile에 나열된 명령어를 단계별로 수행하여 컨테이너 이미지를 작성합니다. Docker는 base부터 시작하여 새 레이어로 변경을 도입하는 각 명령을 실행하는 레이어라는 아이디어를 기반으로 합니다. 각 레이어를 캐시하므로 이미지를 개발하고 다시 빌드할 때 Docker는 수정하지 않은 경우 캐시에서 레이어(종종 중간 레이어라고 함)를 재사용합니다. 편집된 계층에 도달하면 새로운 중간 계층을 빌드하고 이 특정 빌드와 결합시킵니다. 따라서 이미지 재구성과 같은 작업을 매우 효율적으로 수행 할 수 있으며 여러 빌드 버전을 쉽게 유지 관리할 수 있습니다.

    ![Docker Container Image](images/01-container-image.png)

    예를 들어, 초안 파일에서 첫번째 줄인`FROM ubuntu:latest`는 시작점으로 기본 이미지를 지정합니다. 다음 명령인 `RUN apt-get -y update`는 Docker가 Ubuntu 리포지토리에서 패키지 목록을 업데이트하는 새 계층을 만듭니다. 이것은 대부분의 경우 `ENTRYPOINT` *(hint hint)* 또는 실행 파일이 실행되는 마지막 명령어에 도달할 때까지 계속됩니다.

    나머지 명령을 Dockerfile.draft에 추가합니다.

    <details>
    <summary>HINT: Helpful links for completing Dockefile.draft</summary>
    <pre>
    Here are links to external documentation to give you some ideas:

    #[TODO]: Copy the "service" directory into container image

    - Consider the [COPY](https://docs.docker.com/engine/reference/builder/#copy) command
    - You're copying both the python source files and requirements.txt from the "monolith-service/service" directory on your EC2 instance into the working directory of the container, which can be specified as "."

    #[TODO]: Install dependencies listed in the requirements.txt file using pip

    - Consider the [RUN](https://docs.docker.com/engine/reference/builder/#run) command
    - More on [pip and requirements files](https://pip.pypa.io/en/stable/user_guide/#requirements-files)

    #[TODO]: Specify a listening port for the container

    - Consider the [EXPOSE](https://docs.docker.com/engine/reference/builder/#expose) command
    - App listening portNum can be found in the app source - mythicalMysfitsService.py

    #[TODO]: Run "mythicalMysfitsService.py" as the final step. We want this container to run as an executable. Looking at ENTRYPOINT for this?

    - Consider the [ENTRYPOINT](https://docs.docker.com/engine/reference/builder/#entrypoint) command
    - Our ops team typically runs 'python mythicalMysfitsService.py' to launch the application on our servers.
    </pre>
    </details>

    추가한 내용에 만족하거나 혹은 문제가 발생되면 아래 힌트와 비교하여 작업을 확인할 수 있습니다.

    <details>
    <summary>HINT: Completed Dockerfile</summary>
    <pre>
    FROM ubuntu:14.04
    RUN apt-get update -y
    RUN apt-get install -y python-pip python-dev build-essential
    RUN pip install --upgrade pip
    COPY ./service /MythicalMysfitsService
    WORKDIR /MythicalMysfitsService
    RUN pip install -r ./requirements.txt
    EXPOSE 80
    ENTRYPOINT ["python"]
    CMD ["mythicalMysfitsService.py"]
    </pre>
    </details>

    Dockerfile이 잘 만들어졌다면 파일 이름을 "Dockerfile.draft"에서 "Dockerfile"로 바꾸고 다음 단계를 계속합니다.

    <pre>
    $ mv Dockerfile.draft Dockerfile
    </pre>

1. [Docker build](https://docs.docker.com/engine/reference/commandline/build/) 명령을 사용하여 이미지를 빌드합니다.

    이 명령은 Dockerfile이 있는 동일한 디렉토리에서 실행해야합니다. **뒤의 마침표**는 build 명령이 Dockerfile의 현재 디렉토리를 찾도록 지시합니다.

    <pre>
    $ docker build -t monolith-service .
    </pre>

    Docker가 이미지의 모든 레이어를 빌드할 때 많은 출력이 보여집니다. 도중에 문제가 있으면 빌드 프로세스가 실패하고 중지됩니다(빌드 프로세스가 실패하지 않는 한 빨간색 텍스트 및 경고는 문제가 되지 않습니다). 그렇지 않으면 다음과 같이 빌드 출력 마지막에 성공 메시지가 표시됩니다:

    <pre>
    Step 9/10 : ENTRYPOINT ["python"]
     ---> Running in 7abf5edefb36
    Removing intermediate container 7abf5edefb36
     ---> 653ccee71620
    Step 10/10 : CMD ["mythicalMysfitsService.py"]
     ---> Running in 291edf3d5a6f
    Removing intermediate container 291edf3d5a6f
     ---> a8d2aabc6a7b
    Successfully built a8d2aabc6a7b
    Successfully tagged monolith-service:latest
    </pre>

    *Note: 위와 동일하지는 않지만 비슷한 출력 결과를 볼 수 있습니다.*

    Awesome, your Dockerfile built successfully, but our developer didn't optimize the Dockefile for the microservices effort later.  Since you'll be breaking apart the monolith codebase into microservices, you will be editing the source code (e.g. `mythicalMysfitsService.py`) often and rebuilding this image a few times.  Looking at your existing Dockerfile, what is one thing you can do to improve build times?

    Dockerfile은 성공적으로 빌드되었지만 개발자는 나중에 마이크로 서비스로 만들기 위해 Dockefile을 최적화하지 않았습니다. 모놀리스 코드베이스를 마이크로 서비스로 분리하게 되면 소스 코드(예:`mythicalMysfitsService.py`)를 자주 편집하고 이 이미지를 몇 번 다시 작성하게 됩니다. 기존 Dockerfile에 빌드 시간을 향상시키기 위해 할 수 있는 한 가지는 무엇일까요?

    <details>
    <summary>HINT</summary>
    Remember that Docker tries to be efficient by caching layers that have not changed.  Once change is introduced, Docker will rebuild that layer and all layers after it.

    Edit mythicalMysfitsService.py by adding an arbitrary comment somewhere in the file.  If you're not familiar with Python, [comments](https://docs.python.org/2/tutorial/introduction.html) start with the hash character, '#' and are essentially ignored when the code is interpreted.

    For example, here a comment (`# Author: Mr Bean`) was added before importing the time module:
    <pre>
    # Author: Mr Bean

    import time
    from flask import Flask
    from flask import request
    import json
    import requests
    ....
    </pre>

    Rebuild the image using the 'docker build' command from above and notice Docker references layers from cache, and starts rebuilding layers starting from Step 5, when mythicalMysfitsService.py is copied over since that is where change is first introduced:

    <pre>
    Step 5/10 : COPY ./service /MythicalMysfitsService
     ---> 9ec17281c6f9
    Step 6/10 : WORKDIR /MythicalMysfitsService
     ---> Running in 585701ed4a39
    Removing intermediate container 585701ed4a39
     ---> f24fe4e69d88
    Step 7/10 : RUN pip install -r ./requirements.txt
     ---> Running in 1c878073d631
    Collecting Flask==0.12.2 (from -r ./requirements.txt (line 1))
    </pre>

    Try reordering the instructions in your Dockerfile to copy the monolith code over after the requirements are installed.  The thinking here is that the Python source will see more changes than the dependencies noted in requirements.txt, so why rebuild requirements every time when we can just have it be another cached layer.
    </details>

    빌드 시간을 향상시킬 것이라고 생각되도록 Dockerfile을 편집하고 아래의 Final Dockerfile 힌트와 비교해보세요.


    #### Final Dockerfile
    <details>
    <summary>HINT: Final Dockerfile</summary>
    <pre>
    FROM ubuntu:14.04
    RUN apt-get update -y
    RUN apt-get install -y python-pip python-dev build-essential
    RUN pip install --upgrade pip
    COPY service/requirements.txt .
    RUN pip install -r ./requirements.txt
    COPY ./service /MythicalMysfitsService
    WORKDIR /MythicalMysfitsService
    EXPOSE 80
    ENTRYPOINT ["python"]
    CMD ["mythicalMysfitsService.py"]
    </pre>
    </details>

    최적화의 이점을 보려면 먼저 새로운 Dockerfile을 사용하여 모놀리스 이미지를 다시 빌드해야합니다 (5단계 시작시 동일한 빌드 명령 사용). 그런 다음 'mythicalMysfitsService.py'에 변경 사항을 적용하고 (예: 임의의 다른 주석 추가) 모놀리스 이미지를 다시 빌드합니다. Docker는 재정렬 후 첫 번째 빌드에서 requirements를 캐시하고 이 두 번째 빌드에서 캐시를 참조합니다. 아래와 유사한 결과를 볼 수 있습니다:

    <pre>
    Step 6/11 : RUN pip install -r ./requirements.txt
     ---> Using cache
     ---> 612509a7a675
    Step 7/11 : COPY ./service /MythicalMysfitsService
     ---> c44c0cf7e04f
    Step 8/11 : WORKDIR /MythicalMysfitsService
     ---> Running in 8f634cb16820
    Removing intermediate container 8f634cb16820
     ---> 31541db77ed1
    Step 9/11 : EXPOSE 80
     ---> Running in 02a15348cd83
    Removing intermediate container 02a15348cd83
     ---> 6fd52da27f84
    </pre>

    Docker 이미지가 만들어졌습니다. -t 플래그는 컨테이너 이미지의 이름을 지정합니다. docker 이미지를 조회하면 목록에 "monolith-service" 이미지가 표시됩니다. 다음은 샘플 출력입니다. 목록에서 모놀리스 이미지를 참고하세요:

    <pre>
    $ docker images
    REPOSITORY                                                              TAG                 IMAGE ID            CREATED              SIZE
    monolith-service                                                        latest              29f339b7d63f        About a minute ago   506MB
    ubuntu                                                                  latest              ea4c82dcd15a        4 weeks ago          85.8MB
    golang                                                                  1.9                 ef89ef5c42a9        4 months ago         750MB
    </pre>

    *Note: 출력 결과는 동일하지 않지만 비슷할 것입니다.*

    이미지에는 "latest" 태그가 붙어 있습니다. 직접 태그를 지정하지 않은 경우의 기본 설정이며 이를 이미지를 식별하는 자유로운 방식으로 사용할 수 있습니다(예: monolith-service:1.2 또는 monolith-service:experimental). 이미지를 식별하고 이미지를 코드의 branch/version과 연관시키는 데 매우 편리한 방법입니다.

2. 도커 컨테이너를 실행하고 컨테이너로 실행되는 입양 대행사 플랫폼을 테스트합니다:

    [docker run](https://docs.docker.com/engine/reference/run/) 명령을 사용하여 이미지를 실행합니다. -p 플래그는 호스트 수신 포트를 컨테이너 수신 포트에 맵핑하는 데 사용됩니다.

    <pre>
    $ docker run -p 8000:80 -e AWS_DEFAULT_REGION=<b><i>REGION</i></b> -e DDB_TABLE_NAME=<b><i>TABLE_NAME</i></b> monolith-service
    </pre>

    *Note: CloudFormation 스택의 출력에서 만들어진 `workshop-1/cfn-output.json` 파일에서 DynamoDB 테이블 이름을 찾을 수 있습니다.*

    애플리케이션이 시작될 때의 샘플 출력은 다음과 같습니다.:

    ```
    * Running on http://0.0.0.0:80/ (Press CTRL+C to quit)
    ```

    *Note: 출력 결과는 동일하지 않지만 비슷할 것입니다.*

    모놀리스 서비스의 기본 기능을 테스트하기 위해 Cloud9에 번들로 제공되는 [cURL] (https://curl.haxx.se/)과 같은 유틸리티를 사용하여 서비스를 쿼리합니다.

    탭 옆에 있는 + 기호를 클릭하고 **New Terminal**을 선택하거나 Cloud9 메뉴에서 **Window** -> **New Terminal**을 클릭하여 새로운 shell 세션을 열어 다음 curl 명령을 실행합니다.

    <pre>
    $ curl http://localhost:8000/mysfits
    </pre>

    다양한 Mythical Mysfits에 대한 데이터가 포함된 JSON 배열이 표시됩니다.

    *Note: Docker 컨테이너 내에서 실행되는 프로세스는 초기 설정 스크립트에서 Cloud9 환경에 연결된 인스턴스 프로파일의 자격 증명을 검색하기 위해`169.254.169.254`에서 실행되는 EC2 메타 데이터 API 엔드 포인트에 접근할 수 있기 때문에 DynamoDB에 인증할 수 있습니다. 컨테이너의 프로세스는 (컨테이너에 명시적으로 마운트되지 않은 경우)호스트 파일 시스템의`~ /.aws/credentials` 파일에 액세스 할 수 없습니다.*

    모노리스 컨테이너를 실행중인 원래 shell 탭으로 다시 전환하여 모노리스의 출력을 확인합니다.

    The monolith container runs in the foreground with stdout/stderr printing to the screen, so when the request is received, you should see a `200`. "OK".

    모노리스 컨테이너는 stdout/stderr 화면 출력으로 포그라운드에서 실행하므로 요청이 수신되면 '200'. "OK"가 표시됩니다.

    다음은 샘플 출력입니다:

    <pre>
    INFO:werkzeug:172.17.0.1 - - [16/Nov/2018 22:24:18] "GET /mysfits HTTP/1.1" 200 -
    </pre>

    In the tab you have the running container, type **Ctrl-C** to stop the running container.  Notice, the container ran in the foreground with stdout/stderr printing to the console.  In a production environment, you would run your containers in the background and configure some logging destination.  We'll worry about logging later, but you can try running the container in the background using the -d flag.

    실행 컨테이너가 있는 탭에서 **Ctrl-C**를 입력하여 실행 컨테이너를 중지합니다. 컨테이너는 콘솔에 stdout/stderr 출력으로 포그라운드에서 실행되었습니다. 프로덕션 환경에서는 백그라운드에서 컨테이너를 실행하고 로그 저장 경로를 구성합니다. 로깅은 나중에 얘기할 것이므로 -d 플래그를 사용하여 백그라운드에서 컨테이너를 실행합니다.

    <pre>
    $ docker run -d -p 8000:80 -e AWS_DEFAULT_REGION=<b><i>REGION</i></b> -e DDB_TABLE_NAME=<b><i>TABLE_NAME</i></b> monolith-service
    </pre>

    List running docker containers with the [docker ps](https://docs.docker.com/engine/reference/commandline/ps/) command to make sure the monolith is running.

    [docker ps](https://docs.docker.com/engine/reference/commandline/ps/) 명령으로 실행중인 도커 컨테이너에서 모노리스가 실행 중인지 확인합니다.

    <pre>
    $ docker ps
    </pre>

    목록에서 모놀리스가 실행되고 있는 것을 볼 수 있습니다. 이전과 동일한 curl 명령을 반복하여 동일한 Mysfit 목록을 확인합니다. [docker logs](https://docs.docker.com/engine/reference/commandline/ps/)를 실행하여 로그를 다시 확인할 수 있습니다 (컨테이너 이름 또는 id fragment를 인수로 사용).

    <pre>
    $ docker logs <b><i>CONTAINER_ID</i></b></pre>

    위 명령의 샘플 출력은 다음과 같습니다:

    <pre>
    $ docker run -d -p 8000:80 -e AWS_DEFAULT_REGION=<b><i>REGION</i></b> -e DDB_TABLE_NAME=<b><i>TABLE_NAME</i></b> monolith-service
    51aba5103ab9df25c08c18e9cecf540592dcc67d3393ad192ebeda6e872f8e7a
    $ docker ps
    CONTAINER ID        IMAGE                           COMMAND                  CREATED             STATUS              PORTS                  NAMES
    51aba5103ab9        monolith-service:latest         "python mythicalMysf…"   24 seconds ago      Up 23 seconds       0.0.0.0:8000->80/tcp   awesome_varahamihira
    $ curl localhost:8000/mysfits
    {"mysfits": [...]}
    $ docker logs 51a
     * Running on http://0.0.0.0:80/ (Press CTRL+C to quit)
    172.17.0.1 - - [16/Nov/2018 22:56:03] "GET /mysfits HTTP/1.1" 200 -
    INFO:werkzeug:172.17.0.1 - - [16/Nov/2018 22:56:03] "GET /mysfits HTTP/1.1" 200 -</pre>

    위의 샘플 출력에서 컨테이너에는 "awesome_varahamihira"라는 이름이 지정되었습니다. 이름은 임의로 할당됩니다. 실행 이름을 지정하려는 경우 docker run 명령에 이름 옵션을 전달할 수도 있습니다. 자세한 내용은 [Docker run reference] (https://docs.docker.com/engine/reference/run/)에서 확인할 수 있습니다. `docker kill`을 사용하여 컨테이너를 kill합니다.

3. [Elastic Container Registry (ECR)](https://aws.amazon.com/ecr/)에 동작중인 Docker 이미지를 태그하고 푸쉬합니다. ECR은 Docker 컨테이너 이미지를 쉽게 저장, 관리 및 배포 할 수 있는 완전히 관리되는 Docker 컨테이너 레지스트리입니다. 다음 실습에서는 ECS를 사용하여 ECR에서 이미지를 가져옵니다.

    In the AWS Management Console, navigate to [Repositories](https://console.aws.amazon.com/ecs/home#/repositories) in the ECS dashboard.  You should see repositories for the monolith service and like service.  These were created by CloudFormation and named like <code><b><i>STACK_NAME</i></b>-mono-xxx</code> and <code><b><i>STACK_NAME</i></b>-like-xxx</code> where ***STACK_NAME*** is the name of the CloudFormation stack (the stack name may be truncated).

    AWS Management Console에서 ECS 대시보드의 [Repositories] (https://console.aws.amazon.com/ecs/home#/repositories)로 이동합니다. 모놀리스 서비스 및 like 서비스에 대한 저장소가 표시되어야합니다. CloudFormation에 의해 생성되어 <code><b><i>STACK_NAME</i></b>-mono-xxx</code>나 <code><b><i>STACK_NAME</i></b>-like-xxx</code>같이 명명됩니다. 여기서 ***STACK_NAME***은 CloudFormation 스택의 이름입니다(스택 이름이 잘릴 수 있음).

    ![ECR repositories](images/01-ecr-repo.png)

    모놀리스의 리포지토리 이름을 클릭하고 리포지토리 URI를 기록해 둡니다(다음 실습에서 이 값을 다시 사용합니다):

    ![ECR monolith repo](images/01-ecr-repo-uri.png)

    *Note: 저장소 URI는 고유한 값을 가집니다.*

    컨테이너 이미지에 태그를 지정하고 모노리스 저장소에 푸쉬합니다.

    <pre>
    $ docker tag monolith-service:latest <b><i>ECR_REPOSITORY_URI</i></b>:latest
    $ docker push <b><i>ECR_REPOSITORY_URI</i></b>:latest</pre>

    push 명령을 실행하면 Docker는 레이어를 ECR로 푸쉬합니다.

    다음은 명령의 샘플 출력입니다:

    <pre>
    $ docker tag monolith-service:latest 873896820536.dkr.ecr.us-east-2.amazonaws.com/mysfit-mono-oa55rnsdnaud:latest
    $ docker push 873896820536.dkr.ecr.us-east-2.amazonaws.com/mysfit-mono-oa55rnsdnaud:latest
    The push refers to a repository [873896820536.dkr.ecr.us-east-2.amazonaws.com/mysfit-mono-oa55rnsdnaud:latest]
    0f03d692d842: Pushed
    ddca409d6822: Pushed
    d779004749f3: Pushed
    4008f6d92478: Pushed
    e0c4f058a955: Pushed
    7e33b38be0e9: Pushed
    b9c7536f9dd8: Pushed
    43a02097083b: Pushed
    59e73cf39f38: Pushed
    31df331e1f23: Pushed
    630730f8c75d: Pushed
    827cd1db9e95: Pushed
    e6e107f1da2f: Pushed
    c41b9462ea4b: Pushed
    latest: digest: sha256:a27cb7c6ad7a62fccc3d56dfe037581d314bd8bd0d73a9a8106d979ac54b76ca size: 3252</pre>

    *Note: Typically, you'd have to log into your ECR repo. However, you did not need to authenticate docker with ECR because the [Amazon ECR Credential Helper](https://github.com/awslabs/amazon-ecr-credential-helper) has been installed and configured for you on the Cloud9 Environment.  This was done earlier when you ran the setup script. You can read more about the credentials helper in this [article](https://aws.amazon.com/blogs/compute/authenticating-amazon-ecr-repositories-for-docker-cli-with-credential-helper/).*

일반적으로 ECR 리포지토리에 로그인해야합니다. 그러나 Cloud9 환경에 [Amazon ECR Credential Helper](https://github.com/awslabs/amazon-ecr-credential-helper)가 설치되고 구성되어 ECR에 도커를 인증할 필요가 없습니다. 설정 스크립트를 실행할 때 이전에 수행되었습니다. 자격 증명 도우미에 대한 자세한 내용은 다음 [문서](https://aws.amazon.com/blogs/compute/authenticating-amazon-ecr-repositories-for-docker-cli-with-credential-helper/)에서 확인할 수 있습니다.

    콘솔에서 ECR 리포지토리 페이지를 새로 고치면 latest로 태그되고 업로드된 새 이미지가 표시됩니다.

    ![ECR push complete](images/01-ecr-push-complete.png)

### Checkpoint:
이 시점에서는 ECR 리포지토리에 저장된 모놀리스 코드베이스용 작업 컨테이너가 있어야하며 다음 실습에서 ECS와 함께 배포할 수 있습니다.

[*^ back to the top*](#monolith-to-microservices-with-docker-and-aws-fargate)

## Lab 2 - Deploy your container using ECR/ECS

개별 컨테이너를 배포하는 것은 어렵지 않습니다. 그러나 많은 컨테이너 배포를 조정해야하는 경우 ECS와 같은 컨테이너 관리 도구를 사용하면 작업을 크게 단순화시킬 수 있습니다.

ECS는 애플리케이션 또는 서비스를 구성하는 하나 이상의 컨테이너를 설명하는 [Task Definition](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definitions.html)라는 JSON 형식의 템플릿을 참조합니다. 작업 정의(Task Definition)는 ECS가 EC2 인스턴스 또는 AWS Fargate에서 컨테이너를 **task**로 실행하기 위해 사용하는 레시피입니다.

<details>
<summary>INFO: What is a task?</summary>
A task is a running set of containers on a single host. You may hear or see 'task' and 'container' used interchangeably. Often, we refer to tasks instead of containers because a task is the unit of work that ECS launches and manages on your cluster. A task can be a single container, or multiple containers that run together.

Fun fact: a task is very similar to a Kubernetes 'pod'.
</details>

대부분의 작업 정의 매개 변수는 [docker run](https://docs.docker.com/engine/reference/run/) 명령에 전달된 옵션 및 인수에 매핑됩니다. 이는 사용할 컨테이너 이미지, host:container 포트 매핑, CPU 및 메모리 할당, 로깅 등과 같은 구성을 설명할 수 있음을 의미합니다.

이 실습에서는 ECS와 함께 ECR에 저장된 컨테이너화된 입양 플랫폼을 배포하기 위한 기반으로 제공되는 작업 정의를 만듭니다. 서버 또는 기타 인프라를 관리하지 않고도 컨테이너를 실행할 수있는 [Fargate](https://aws.amazon.com/fargate/) 시작 유형을 사용하게됩니다. Fargate 컨테이너는 ECS 작업에 EC2 인스턴스의 동일한 네트워킹 속성을 제공하는 [awsvpc](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-networking.html)라는 네트워킹 모드로 시작됩니다. 작업에는 기본적으로 자체 [elastic network interface](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html)가 제공됩니다. 이는 작업별 보안 그룹과 같은 이점을 제공합니다. 이제 시작해봅니다!

![Lab 2 Architecture](images/02-arch.png)

*Note: 이 랩에서는 AWS Management Console을 사용하지만 AWS CLI, SDK 또는 CloudFormation을 사용하여 프로그래밍 방식으로 동일한 작업을 수행할 수 있습니다.*

### Instructions:

1. 모놀리스를 실행하는 데 필요한 것을 설명하는 ECS 작업 정의를 만듭니다.

    워크숍 초반에 실행한 CloudFormation 템플릿은 간단한 "Hello World" NGINX 컨테이너를 실행하는 placeholder용 ECS 리소스를 만들었습니다(`cfn-output.json`에서 제공되는 CloudFormation에 의해 생성된 ALB에 대한 퍼블릭 엔드 포인트에서 현재 실행중인 것을 볼 수 있습니다). 이전 실습에서 빌드된 컨테이너 참조하는 새로운 "작업 정의"를 생성하여 모놀리스를 실행하도록 이 placeholder용 인프라를 사용합니다.

    AWS Management Console에서 ECS 대시보드의 [작업 정의](https://console.aws.amazon.com/ecs/home#/taskDefinitions)로 이동합니다. <code>Monolith-Definition-<b><i>STACK_NAME</i></b></code>라는 작업 정의를 찾아서 선택한 다음 "새 개정 생성"을 클릭합니다. "Container Definitions"에서 "monolith-service" 컨테이너를 선택하고 방금 ECR로 푸쉬한 모놀리스 컨테이너의 이미지 URI를 가리키도록 "Image"를 업데이트합니다(`018782361163.dkr.ecr.us-east-1.amazonaws.com/mysfit-mono-oa55rnsdnaud:latest` 같은 형식).

    ![Edit container example](images/02-task-def-edit-container.png)

2. 컨테이너 정의에서 CloudWatch 로깅 설정을 확인합니다.

    이전 실습에서는 *stdout*을 보기 위해 실행중인 컨테이너에 연결했지만 프로덕션 환경에서는 이런 방식이 아닌 중앙 로깅 솔루션을 구현하는 것이 좋은 운영 방식입니다. ECS는 컨테이너 정의에서 활성화할 수 있는 awslogs 드라이버를 통해 [CloudWatch logs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/WhatIsCloudWatchLogs.html)와의 통합을 제공합니다.

    "스토리지 및 로깅"에서 "로그 드라이버"가 "awslogs"로 설정되어 있는지 확인합니다.

    로그 구성은 다음과 같습니다:

    ![CloudWatch Logs integration](images/02-awslogs.png)

    "업데이트"를 클릭하여 컨테이너 설정을 저장 한 다음 "생성"을 클릭하여 작업 정의 새 버전을 만듭니다.

3. [작업 실행](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs_run_task.html)을 사용하여 작업 정의를 실행합니다.

    아래와 같은 작업 정의보기에서 새 개정을 작성하거나 특정 조치를 하는 것과 같은 작업을 수행합니다. **Action** 드롭 다운에서 **Run Task**을 선택하여 컨테이너를 시작합니다.

    ![Run Task](images/02-run-task.png)

    다음 필드를 구성합니다:

    * **Launch Type** - **Fargate** 선택
    * **Cluster** - 드롭 다운 메뉴에서 워크샵 클러스터를 선택
    * **Task Definition** - 드롭 다운 메뉴에서 생성한 작업 정의를 선택

    "VPC 및 보안 그룹" 섹션에서 다음을 입력:

    * **Cluster VPC** - 워크샵 VPC, <code>Mysfits-VPC-<b><i>STACK_NAME</i></b></code>
    * **Subnets** - <code>Mysfits-PublicOne-<b><i>STACK_NAME</i></b></code> 같은 Public Subnet 선택
    * **Security goups** - 기본값을 사용하지만 포트 80에서 인바운드 트래픽을 허용하는지 확인
    * **Auto-assign public IP** - "ENABLED"

    나머지 필드는 모두 기본값으로 두고 **Run Task**를 클릭합니다.

    작업이 **PENDING** 상태에서 시작되는 것을 볼 수 있습니다(placeholder NGINX 작업도 여전히 실행중).

    ![Task state](images/02-task-pending.png)

    몇 초 안에 작업이 **RUNNING** 상태로 바뀔 때까지 새로 고침 버튼을 클릭합니다.

    ![Task state](images/02-task-running.png)

4. Cloud9 환경의 cURL을 사용하여 간단한 GET 요청을 보내 실행중인 작업을 테스트합니다.

    먼저 작업의 IP를 결정해야합니다. "Fargate" 시작 유형을 사용하면 각 작업에 고유한 ENI 및 Public/Private IP 주소가 부여됩니다. 방금 시작한 작업의 ID를 클릭하여 작업의 세부 정보 페이지로 이동하십시오. curl 명령에 사용할 Public IP 주소를 기록해 둡니다.

    ![Container Instance IP](images/02-public-IP.png)

    이전과 동일한 curl 명령을 실행하거나 브라우저에서 엔드 포인트를 조회하여 응답에 Mysfits 목록이 보이는지 확인합니다.

    <details>
    <summary>HINT: curl refresher</summary>
    <pre>
    $ curl http://<b><i>TASK_PUBLIC_IP_ADDRESS</i></b>/mysfits
    </pre>
    </details>

    Navigate to the [CloudWatch Logs dashboard](https://console.aws.amazon.com/cloudwatch/home#logs:), and click on the monolith log group (e.g.: `mysfits-MythicalMonolithLogGroup-LVZJ0H2I2N4`).  Logging statements are written to log streams within the log group.  Click on the most recent log stream to view the logs.  The output should look very familiar from your testing in Lab 1.

    [CloudWatch Logs dashboard](https://console.aws.amazon.com/cloudwatch/home#logs:)로 이동하고 모놀리스 로그 그룹(예:`mysfits-MythicalMonolithLogGroup-LVZJ0H2I2N4`)을 클릭합니다. 로그는 로그 그룹 내의 로그 스트림에 기록됩니다. 가장 최근의 로그 스트림을 클릭하여 로그를 조회합니다. 실습1의 테스트와 매우 친숙한 결과가 보입니다.

    ![CloudWatch Log Entries](images/02-cloudwatch-logs.png)

    If the curl command was successful, stop the task by going to your cluster, select the **Tasks** tab, select the running monolith task, and click **Stop**.

    curl 명령이 성공한 경우 클러스터로 이동하고 **Tasks** 탭을 선택한 후 실행중인 모놀리스 작업을 선택한 다음 **Stop**를 클릭하여 작업을 중지합니다.

### Checkpoint:
잘 하셨습니다! 작업 정의를 만들었으며 ECS를 사용하여 모놀리스 컨테이너를 배포할 수 있습니다. CloudWatch Logs에 로깅을 활성화하여 컨테이너가 예상대로 작동하는지 확인할 수 있습니다.

[*^ back to the top*](#monolith-to-microservices-with-docker-and-aws-fargate)

## Lab 3 - Scale the adoption platform monolith with an ALB

마지막 실습에서 사용한 Run Task 방법은 테스트에는 적합하지만 입양 플랫폼은 장기 실행 프로세스(long running process)로 실행해야합니다.

이 실습에서는 Elastic Load Balancing [Appliction Load Balancer (ALB)](https://aws.amazon.com/elasticloadbalancing/)을 사용하여 들어오는 요청을 실행중인 컨테이너에 배포합니다. 단순한 로드 밸런싱 외에도 다른 서비스에 대한 경로 기반 라우팅(path-based routing)과 같은 기능이 제공됩니다.

이 모두를 묶는 것은 원하는 작업 수(즉, n개의 컨테이너를 장기 실행 프로세스로 유지)를 유지하고 ALB와 통합하는(즉, ALB에 컨테이너의 등록/해제를 처리) **ECS Service**입니다. 워크숍 시작시 CloudFormation에서 초기 ECS 서비스 및 ALB를 생성했습니다. 이 실습에서는 컨테이너화된 모놀리스 서비스를 호스팅하기 위해 해당 리소스를 업데이트합니다. 나중에 모놀리스를 분리할 때는 처음부터 새로운 서비스를 만들게 됩니다.

![Lab 3 Architecture](images/03-arch.png)

### Instructions:

1. placeholder 서비스 테스트:

    워크숍 초반에 시작한 CloudFormation 스택에는 NGINX 웹 서버와 함께 간단한 컨테이너를 실행하는 placeholder ECS 서비스 앞에 ALB가 포함되었습니다. `cfn-output.json` 파일의 "LoadBalancerDNS" 출력 변수에서 이 ALB의 호스트 이름을 찾아 NGINX 기본 페이지를 로드할 수 있는지 확인합니다:

    ![NGINX default page](images/03-nginx.png)


2. Task Definition을 사용하도록 서비스 업데이트:

    <code>Cluster-<i><b>STACK_NAME</b></i></code>라는 ECS 클러스터를 찾은 다음 <code><b><i>STACK_NAME</i></b>-MythicalMonolithService-XXX</code>라는 서비스를 클릭하고 오른쪽 상단의 "Update"를 클릭합니다.

    ![update service](images/03-update-service.png)

    Task Definition를 이전 실습에서 생성한 Revision으로 업데이트한 다음 나머지 화면을 클릭하고 서비스를 업데이트합니다.

3. 웹 사이트 기능 테스트:

    서비스 페이지의 "Tasks"탭에서 배포 진행 상황을 모니터링 할 수 있습니다:

    ![monitoring the update](images/03-deployment.png)

    최신 revision을 실행하는 작업 인스턴스가 하나만 있다면 업데이트가 완전히 배포된 것입니다:

    ![fully deployed](images/03-fully-deployed.png)

    Mythical Mysfits(이전에는 비어 있음)의 S3 정적 사이트를 방문하면 업데이트가 완전히 배포되어 이제 Mysfits로 채워진 페이지가 표시됩니다. 버킷 이름을 `workshop-1/cfn-output.json` 파일에서 찾을 수 있는 <code>http://<b><i>BUCKET_NAME</i></b>.s3-website.<b><i>REGION</i></b>.amazonaws.com/</code>에서 웹 사이트에 액세스 할 수 있습니다:

    ![the functional website](images/03-website.png)

    심장 아이콘을 클릭하여 Mysfit을 like한 다음 Mysfit을 클릭하여 자세한 프로파일을 보고 like수가 증가했는지 확인합니다:

    ![like functionality](images/03-like-count.png)

    이를 통해 모놀리스가 DynamoDB에서 읽고 쓸 수 있으며 like를 처리 할 수 있습니다. ECS에서 CloudWatch 로그를 확인하고 로그의 메시지에서 "Like processed."를 볼 수 있는지 확인합니다. 

    ![like logs](images/03-like-processed.png)

<details>
<summary>INFO: What is a service and how does it differ from a task??</summary>

[ECS service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs_services.html)는 ECS를 통해 ECS 클러스터에서 동시에 수행하는 작업 정의 인스턴스의 지정된 수("desired count")를 실행하고 유지하도록 하는 개념입니다.

tl;dr **Service**는 여러 **tasks**으로 구성되어 있으며 작업을 계속 실행하도록 유지합니다. 자세한 내용은 위의 링크를 참조하시기 바랍니다.

</details>

### Checkpoint:
컨테이너화된 Mythical Mysfits 애플리케이션을 관리하는 로드 밸런싱된 ECS 서비스를 만들었습니다. 여전히 단일 모놀리식 컨테이너이므로 이제 분할을 해보도록 합니다.

[*^ back to the top*](#monolith-to-microservices-with-docker-and-aws-fargate)

## Lab 4: Incrementally build and deploy each microservice using Fargate

모놀리식 방식을 마이크로 서비스로 분할해 봅니다. 이를 위해 모놀리스가 어떻게 작동하는지 자세히 살펴 보겠습니다.

> 모놀리스는 다른 경로에서 여러 가지 다른 API 리소스를 제공하여 Mysfit에 대한 정보를 가져오거나 "like" 하고 입양합니다.
>
> 이러한 리소스에 대한 로직은 일반적으로 "processing"(사용자가 특정 작업을 수행 할 수 있도록 하거나 Mysfit을 입양할 수 있도록 하는 등)와  DynamoDB 같은 persistence layer와의 일부 상호 작용으로 구성됩니다.

> 많은 서비스들이 하나의 데이터베이스와 직접 통신하는 것은 종종 좋지 않은 생각입니다(인덱스 추가나 데이터 마이그레이션을 한 응용 프로그램으로 수행하는 것은 어려움). 주어진 자원의 모든 로직을 분리된 서비스로 분리하기보다는 "프로세싱" 비즈니스 로직만 별도의 서비스로 옮기는 것부터 시작하여 모노리스를 데이터베이스에 대한 facade로 사용합니다. 이것은 그림에서 모놀리스를 "strangling"하고 완전히 교체 될 때까지 옮기기 가장 어려운 부분을 계속 사용하도록 하기 때문에 [Strangler Application pattern](https://www.martinfowler.com/bliki/StranglerApplication.html) 이라고도 합니다. 

> ALB에는 [path-based routing](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-listeners.html#path-conditions)이라는 또 다른 기능이 있으며 URL을 기반으로 특정 대상 그룹에 트래픽을 라우팅합니다. 즉, 마이크로 서비스를 호스팅하기 위해 단일 ALB 인스턴스만 있으면 됩니다. 모놀리스 서비스는 기본 경로인 '/'에 대한 모든 트래픽을 수신합니다. 입양 및 like 서비스 경로는 각각 '/adopt'와 '/like'입니다.

구현할 내용은 다음과 같습니다:

![Lab 4](images/04-arch.png)

*Note: 녹색 작업은 모놀리스를 나타내고 주황색 작업은 "like" 마이크로 서비스를 나타냅니다.
    
모놀리스와 마찬가지로 [Fargate](https://aws.amazon.com/fargate/)를 사용하여 마이크로 서비스를 배포하지만 이번에는 새로운 서비스를 위한 모든 배포 단계를 알아보도록 합니다.

### Instructions:

1. 먼저, "like"기능을 별도의 서비스로 옮기는 것을 지원하기 위해 모노리스에 코드를 추가해야합니다. 이를 위해 Cloud9 환경을 사용합니다. 탭을 닫은 경우 [Cloud9 Dashboard](https://console.aws.amazon.com/cloud9/home)로 이동하여 환경을 찾습니다. "**Open IDE**"를 클릭하고 `app/monolith-service/service/mythicalMysfitsService.py` 소스 파일을 찾아서 다음 섹션의 주석 처리를 제거합니다:

    ```
    # @app.route("/mysfits/<mysfit_id>/fulfill-like", methods=['POST'])
    # def fulfillLikeMysfit(mysfit_id):
    #     serviceResponse = mysfitsTableClient.likeMysfit(mysfit_id)
    #     flaskResponse = Response(serviceResponse)
    #     flaskResponse.headers["Content-Type"] = "application/json"
    #     return flaskResponse
    ```

    이는 여전히 DynamoDB에 대한 지속성을 관리할 수 있는 엔드 포인트를 제공하지만 `process_like_request` 에 의해 처리되는 "비즈니스 로직"은 생략합니다(이 경우에는 print 문일 뿐이지만 실제로는 권한 검사 또는 기타 중요한 처리가 필요할 수 있음).

2. 새로운 기능이 모놀리스에 추가되면 `nolike`와 같은 새로운 태그를 사용하여 모놀리스 도커 이미지를 다시 빌드하고 이전과 마찬가지로 ECR로 푸시합니다 (애매모호할 수 있는 `latest` 태그를 피하는 것이 가장 좋습니다. 대신 고유하고 설명적인 이름을 선택하거나 더 나은 사용자 Git SHA나 빌드 ID를 선택합니다)

    <pre>
    $ cd app/monolith-service
    $ docker build -t monolith-service:nolike .
    $ docker tag monolith-service:nolike <b><i>ECR_REPOSITORY_URI</i></b>:nolike
    $ docker push <b><i>ECR_REPOSITORY_URI</i></b>:nolike
    </pre>

3. 이제 실습 2에서와 같이 모놀리식 Task Definition의 새 revision(이번에는 컨테이너 이미지의 "nolike" 버전을 가리킴)을 만들고 실습 3에서와 같이 이 개정판을 사용하도록 모놀리스 서비스를 업데이트합니다.

4. 이제 like 서비스를 빌드하고 ECR로 푸쉬합니다.

    like-service ECR repo URI를 찾으려면 ECS 대시 보드에서 [Repositories](https://console.aws.amazon.com/ecs/home#/repositories)로 이동하여 <code><b><i>STACK_NAME</i></b>-like-XXX</code>를 찾습니다. like 서비스 저장소를 클릭하고 저장소 URI를 복사합니다.

    ![Getting Like Service Repo](images/04-ecr-like.png)

    *Note: URI는 고유한 값을 가집니다.*

    *Note: Dockerfile을 반드시 아래와 같이 변경한 후 빌드합니다.*

    <pre>
    FROM ubuntu:14.04
    RUN echo Updating existing packages, installing and upgrading python and pip.
    RUN apt-get update -y
    RUN apt-get install -y python-pip python-dev build-essential
    RUN pip install --upgrade pip
    RUN echo Copying the Mythical Mysfits Flask service into a service directory.
    COPY ./service /MythicalMysfitsService
    WORKDIR /MythicalMysfitsService
    RUN echo Installing Python packages listed in requirements.txt
    COPY service/requirements.txt .
    RUN pip install --ignore-installed urllib3 -r ./requirements.txt
    RUN echo Starting python and starting the Flask service...
    ENTRYPOINT ["python"]
    CMD ["mysfits_like.py"]
    </pre>

    <pre>
    $ cd app/like-service
    $ docker build -t like-service .
    $ docker tag like-service:latest <b><i>ECR_REPOSITORY_URI</i></b>:latest
    $ docker push <b><i>ECR_REPOSITORY_URI</i></b>:latest
    </pre>

5. ECR로 푸쉬된 이미지를 사용하여 like 서비스에 대한 새로운 **Task Definition**를 만듭니다.

    ECS 대시 보드에서 [Task Definitions](https://console.aws.amazon.com/ecs/home#/taskDefinitions)로 이동합니다. **Create New Task Definition**을 클릭합니다.

    **Fargate** launch type을 선택하고 **Next step**을 클릭합니다.

    Task Definition의 이름을 입력합니다(예: mysfits-like).

    "[Task execution IAM role](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_execution_IAM_role.html)" 섹션에서 Fargate는 컨테이너 이미지를 가져오고 CloudWatch에 로깅할 수 있는 IAM 역할이 필요합니다. 모놀리스 서비스용으로 이미 생성된 <code><b><i>STACK_NAME</i></b>-EcsServiceRole-XXXXX</code>와 같은 역할을 선택합니다.

    "[Task size](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definition_parameters.html#task_size)" 섹션에서 작업에 사용될 CPU 및 메모리를 지정할 수 있습니다. 이는 컨테이너별 CPU 및 메모리 값과 다르며 컨테이너 정의를 추가할 때 구성 할 수도 있습니다.

    **Task memory (GB)**는 **0.5GB**로, **Task CPU (vCPU)**는 **0.25vCPU**로 선택합니다.

    다음과 같습니다:

    ![Fargate Task Definition](images/04-taskdef.png)

    **Add container**를 클릭하고 like 컨테이너를 작업과 연결합니다.
    
    다음 필드에 값을 입력합니다:

    * **Container name** - 컨테이너 이미지의 이름이 아닌 논리적 식별자입니다(예: `mysfits-like`).
    * **Image** - ECR에 저장된 컨테이너 이미지에 대한 참조입니다. 형식은 like 서비스 컨테이너를 ECR로 푸쉬하는 데 사용한 값과 동일해야합니다. <pre><b><i>ECR_REPOSITORY_URI</i></b>:latest</pre>
    * **Port mapping** - 컨테이너 포트 `80`.

    예를 들어 아래와 같습니다:

    ![Fargate like service container definition](images/04-containerdef.png)

    *Note: Fargate는 awsvpc 네트워크 모드를 사용하므로 호스트 포트를 지정할 필요가 없습니다. 시작 유형 (EC2 또는 Fargate)에 따라 일부 작업 정의 매개 변수가 필요하고 일부는 선택 사항입니다. [task definition documentation](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definitions.html)에서 자세한 내용을 확인할 수 있습니다.*

    like 서비스 코드는 데이터를 DynamoDB에 유지하기 위해 모놀리스에 엔드 포인트를 호출하도록 설계되었습니다. fulfillment를 보낼 위치를 알기 위해 `MONOLITH_URL`이라는 환경 변수를 참조합니다.

    "Advanced container configuration" 섹션으로 아래로 스크롤하고 "Environment"섹션에서 키로 `MONOLITH_URL`을 사용하여 환경 변수를 생성합니다. 값으로는 현재 모노리스 앞에 있는 **ALB DNS name**를 입력합니다.

    예를 들면 다음과 같습니다("http" 또는 슬래시 없이 `alb-mysfits-1892029901.eu-west-1.elb.amazonaws.com`과 같은 호스트 이름만 입력):
.
    ![monolith env var](images/04-env-var.png)

    Fargate를 사용하면 CloudWatch에 편리하게 로깅할 수 있습니다. 기본 로그 설정을 유지하고 **awslogs-group** 및 **awslogs-stream-prefix**를 기록하여 나중에 이 작업에 대한 로그를 찾을 수 있습니다.

    예를 들어 아래와 같습니다:

    ![Fargate logging](images/04-logging.png)

    켄테이너 정의를 연결시키려면 **Add**, and Task Definition을 생성하려면 **Create**를 클릭합니다.

1. 방금 작성한 Like Service 태스크 정의를 실행하도록 ECS 서비스를 생성하고 기존 ALB와 연결 시킵니다.

    방금 작성한 Like 작업 정의의 새 revision으로 이동합니다. **Actions** 드롭 다운에서 **Create Service**를 선택합니다.

    Configure the following fields:

    * **Launch type** - **Fargate** 선택
    * **Cluster** - 워크샵 ECS cluster 선택
    * **Service name** - 서비스 이름 입력(예: `mysfits-like-service`)
    * **Number of tasks** - `1` 입력

    예를 들어 아래와 같습니다:

    ![ECS Service](images/04-ecs-service-step1.png)

    다른 설정은 기본값으로 두고 **Next Step**를 클릭합니다.

    작업 정의는 awsvpc 네트워크 모드를 사용하므로 작업을 호스팅할 VPC 및 서브넷을 선택할 수 있습니다.

    **Cluster VPC**의 경우 워크샵 VPC를 선택합니다. **Subnets**의 경우 프라이빗 서브넷을 선택합니다. 태그로 이를 식별할 수 있습니다.

    인바운드 포트 80을 허용하는 기본 보안 그룹을 그대로 둡니다. VPC에 자체 보안 그룹이 정의되어 있으면 여기에 할당 할 수 있습니다.

    예를 들어 아래와 같습니다:

    ![ECS Service VPC](images/04-ecs-service-vpc.png)

    "Load balancing"으로 아래로 스크롤하여 *Load balancer type*에 대해 **Application Load Balancer**를 선택합니다.

    **Load balancer name** 드롭 다운 메뉴가 나타납니다. 모놀리스 ECS 서비스에 사용된 것과 동일한 Mythical Mysfits ALB를 선택합니다.

    "Container to load balance"섹션의 드롭 다운 메뉴에서 like 서비스 태스크 정의에 해당하는 **Container name : port** 콤보를 선택합니다.

    아래와 비슷하게 구성될 것입니다:

    ![ECS Load Balancing](images/04-ecs-service-alb.png)

    더 많은 설정을 보려면 **Add to load balancer**를 클릭합니다.

    **Production listener Port**의 경우 드롭 다운에서 **80:HTTP**를 선택합니다.

    **Target Group Name**의 경우 Like 컨테이너에 대한 새 그룹을 작성해야하므로 "create new"으로 남겨두고 자동 생성된 값을 `mysfits-like`로 바꿉니다. 이는 대상 그룹을 식별하기 위한 이름이므로 Like 마이크로 서비스와 관련된 값을 사용합니다.

    경로 패턴을 `/mysfits/*/like`로 변경합니다. ALB는이 경로를 사용하여 트래픽을 like 서비스 대상 그룹으로 라우팅합니다. 이것이 동일한 ALB 리스너에서 여러 서비스를 제공하는 방식입니다. 기존 기본 경로는 모놀리스 대상 그룹으로 라우팅됩니다.

    **Evaluation order**에는 `1`을 입력합니다.  **Health check path**은 `/`가 되게 합니다.

    마지막으로 **Enable service discovery integration**을 선택 취소합니다. public namespaces가 지원되지만 Route53에서 먼저 public zone을 구성해야합니다. 서비스에 대해 이 편리한 기능을 고려중이라면 [service discovery](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-discovery.html)에서 자세한 내용을 읽을 수 있습니다.

    설정은 아래와 같을 것입니다:

    ![Like Service](images/04-ecs-service-alb-detail.png)

    다른 필드는 기본값으로 두고 **Next Step**를 클릭합니다.

    **Next Step**를 클릭하여 Auto Scaling 구성은 건너 뜁니다.

    Review 페이지에서 **Create Service**을 클릭합니다.

    서비스가 생성되면 **View Service**를 클릭하여 작업 정의가 서비스로 배포된 것을 볼 수 있습니다. **PROVISIONING** 상태에서 시작하여 **PENDING** 상태로 진행하며 구성에 성공하면 서비스는 **RUNNING** 상태가됩니다. 새로 고침 버튼을 주기적으로 클릭하면 이러한 상태 변화를 볼 수 있습니다.

2. 새로운 like 서비스가 배포되면 웹 사이트를 방문하여 Mysfit을 다시 테스트합니다. CloudWatch 로그를 다시 확인하고 like 서비스에 "Like processed" 메시지가 표시되는지 확인합니다. 이것이 보이면 like 기능을 마이크로 서비스에 성공적으로 옮긴 것입니다.

3. 시간이 있다면 모놀리스에서 이제 더 이상 사용되지 않는 기존 like 엔드 포인트를 제거 할 수 있습니다.

    모노리스 및 like 서비스 컨테이너 이미지를 빌드한 Cloud9 환경으로 돌아갑니다.

    monolith 폴더에서 mythicalMysfitsService.py를 Cloud9 편집기에서 열고 아래 코드를 찾습니다:

    ```
    # increment the number of likes for the provided mysfit.
    @app.route("/mysfits/<mysfit_id>/like", methods=['POST'])
    def likeMysfit(mysfit_id):
        serviceResponse = mysfitsTableClient.likeMysfit(mysfit_id)
        process_like_request()
        flaskResponse = Response(serviceResponse)
        flaskResponse.headers["Content-Type"] = "application/json"
        return flaskResponse
    ```
    해당 라인을 찾으면 삭제하거나 주석 처리할 수 있습니다.

    *Tip: Python에 익숙하지 않은 경우, 줄의 시작 부분에 해시 문자 "#"를 추가하여 줄을 주석 처리할 수 있습니다. *

4. 모놀리스 이미지를 빌드, 태그하고 모놀리스 ECR 리포지토리에 푸시합니다.

    `nolike` 대신 `nolike2` 태그를 사용합니다.

    <pre>
    $ docker build -t monolith-service:nolike2 .
    $ docker tag monolith-service:nolike2 <b><i>ECR_REPOSITORY_URI</i></b>:nolike2
    $ docker push <b><i>ECR_REPOSITORY_URI</i></b>:nolike2
    </pre>

    ECR에서 모놀리스 저장소를 보면 푸쉬된 이미지에 `nolike2`태그가 붙은 것을 볼 수 있습니다:

    ![ECR nolike image](images/04-ecr-nolike2.png)

5. 새로운 컨테이너 이미지 URI를 참조하도록 모노리스에 대한 마지막 Task Definition를 작성합니다(이 프로세스는 이제 익숙해야하며 프로덕션 환경에서는 CI/CD서비스에 맡기는 것이 합리적임을 알 수 있습니다). 새로운 Task Definition를 사용도록 모놀리스 서비스를 업데이트하고 앱이 여전히 이전과 같이 작동하는지 확인합니다.

### Checkpoint:
축하합니다. 모놀리스에서 like 마이크로 서비스를 성공적으로 생성했습니다. 시간이 있다면 이 실습을 반복하여 입양 마이크로 서비스를 분해합니다. 그렇지 않은 경우 **Workshop Cleanup**에서 워크숍 중에 생성된 모든 자산이 제거되도록 아래 단계를 수행하여 오늘 이후에 예기치 않은 비용이 발생하지 않도록 합니다.

## Workshop Cleanup

계정에 무언가를 두면 요금이 계속 발생하기 때문에 이는 매우 중요합니다. 어떤 것들은 CloudFormation에서 생성하고 어떤 것들은 워크샵을 통해 수동으로 생성했습니다. 아래 단계에 따라 올바르게 삭제합니다.

랩 전체에서 수동으로 생성된 리소스를 삭제합니다:

* ECS service - 먼저 desired task count를 0으로 업데이트한 다음 ECS 서비스 자체를 삭제합니다.
* ECR - ECR 저장소로 푸쉬된 모든 Docker 이미지를 삭제합니다.
* CloudWatch 로그 그룹
* ALB 및 관련 대상 그룹

마지막으로, 워크숍 시작시 시작된 [delete the CloudFormation stack](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-delete-stack.html)가 나머지를 정리합니다. 스택 삭제 프로세스에 오류가 발생하면 CloudFormation 대시 보드에서 이벤트 탭을 보고 실패한 단계를 확인합니다. CloudFormation에서 관리하는 리소스에 연결된 수동으로 생성된 자산을 정리해야하는 경우 일 수 있습니다.