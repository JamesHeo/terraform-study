# 재사용 가능한 모듈 만들어보기

이번 챕터에서는 모듈을 어떻게 효율적으로 활용할 것인지에 대해 알아봅시다.
대부분의 팀에서는 운영환경과 테스트환경을 구분해서 작업을 합니다. 이 때 운영환경을 구축하는데에 활용한 코드를 복사하고 똑같이 붙여넣어서 테스트 환경을 구축하는 것은 비효율 적입니다. 모듈을 활용해서 효율적으로 한 개 이상의 환경을 구축하는 법을 알아봅시다. 

이번 챕터는 하단과 같이 이루어져 있습니다.
* 모듈 기본
* 모듈 입력 변수
* 모듈 지역 변수
* 모듈 출력 변수
* 모듈 문제
* 모듈 버전관리

## 모듈의 기본
모듈의 기본 구성은 하단과 같이 이루어져 있습니다. "NAME" 은 모듈을 식별하는데에 사용이 되며 "SOURCE"는 모듈의 위치를 말해줍니다. 이 둘을 활용하여 테라폼 어디서든 이 모듈을 활용할 수 있습니다.
```json
module "NAME" {
    source = "SOURCE"
    [CONFIG ...]
}
```

stage/services/webserver-cluster 코드를 아래와 같이 모듈화 해봅시다. 
```json
provider "aws" {
    region = "us-east-2"
}
module "webserver_cluster"{
    source = "../../../modules/services/webserver-cluster"
}
```

모듈 코드를 보면 많은 값들이 하드코딩이 된 것을 확인할 수 있습니다. 이 이슈를 해결하기 위해서 모듈의 입력값에 대해서 알아봅시다.

## 모듈의 입력값
전에 배웠던 입력 변수를 활용해서 modules/services/webservices/webserver-cluster/variables.tf 파일에 하단 코드를 작성해 줍니다.
```json
variable "cluster_name" {
  description = "The name to use for all the cluster resources"
  type        = string
}
```

modules/services/webserver-cluster/main.tf 파일에 하드코딩 된 식별자(terraform-asg-example)를 var.cluster_name으로 바꿔서 입력 변수를 모듈에 적용해줍니다.
```json
resource "aws_security_group" "alb" {
    name = "${var.cluster_name}-alb"
}
ingress {
    ...
}
```

다음으로, 이 모듈을 테스트환경에 적용해봅니다.
```json
module "webserver_cluster" {
    source = "../../../modules/services/webserver-cluster"

    cluster_name = "webservers-stage"
}
```

아래와 같이 운영환경에도 적용해줍니다.
```json
module "webserver_cluster" {
    source = "../../../modules/services/webserver-cluster"

    cluster_name = "webservers-prod"
}
```
입력 변수들을 모듈과 환경 사이의 API라고 생각하면 이해가 쉽습니다.

## 모듈 지역 변수
우리가 정의했던 모듈을 살펴보면 포트넘버, 프로토콜, 등이 반복적으로 정의되고 있는 것을 확인 할 수 있습니다. 입력 변수를 활용 할 수 있지만 지역 변수를 활용하여 같은 값의 반복 입력을 최소화하고 유지보수에도 도움이 됩니다.

아래와 같이 지역변수를 모듈 파일에 정의해줍니다.
```json
locals {
    http_port = 80
    any_port = 0
    any_protocol = "-1"
    tcp_protocol = "tcp"
    all_ips = ["0.0.0.0/0"]
}
```
지역 변수 참조는 아래와 같이 합니다.
```json
locals.<NAME>
```
모듈에서의 사용 예시입니다.
```json
resource "aws_lb_listener" "http" {
    load_balancer_arn = aws_lb.example.arn
    port = local.http_port
    protocol = "HTTP" 

    ...
}

resource "aws_security_group" "alb" {
    name = "${var.cluster_name}-alb"

    ingress{
        from_port = local.http_port
        to_port = local.http_port
        protocol = local.tcp_protocol
        cidr_blocks = local.all_ips
    }
    egress{
        from_port = local.any_port
        to_port = local.any_port
        protocol = local.any_protocol
        cidr_blocks = local.all_ips
    }
}
```
지역 변수를 활용하면 코드를 쉽게 읽고 관리할 수 있는 것을 확인할 수 있습니다. 

## 모듈 출력 변수
출력 변수를 활용해서 모듈에서 필요한 변수들을 출력해 봅시다.
아래와 같이 variables.tf 파일에 출력 변수를 정의합니다.
```json
output "asg_name" {
    value = aws_autoscaling_group.example.name
    description = "The name of the Auto Scaling Group"
}
```
아래와 같이 출력합니다.
```json
module.<MODULE_NAME>.<OUTPUT_NAME>

예: module.frontend.asg_name
```

아래와 같이 활용 될 수 있습니다.
```json
resource "aws_autoscaling_schedule" "scale_out_during_business_hourse" {
    scheduled_action_name = "scale-out-during-business-hours"
    min_size = 2
    max_size = 10
    desired_capacity = 10
    recurrence = "0 9 * * *"

    autoscaling_group_name = module.webserver_cluster.asg_name
}
```

## 모듈 문제
모듈을 작성할 때 주의할 점들이 있습니다.
* 파일 경로
* 인라인 블록