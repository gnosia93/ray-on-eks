# ray-on-eks

### Request Details and Expected Outcome ###

Who builds ML models for their end customers and hosts them using Ray on EKS. They are looking for guidance on,

* Best practices for hosting ML models on Ray on EKS — including scaling, resource management, and production-readiness.
* Cost tracking across the ML model lifecycle — how to attribute and monitor costs per model/customer across training, inference, and iteration phases.


### 🏗️ 랩 아키텍처 개요 ###
*	인프라: EKS (Karpenter를 통한 가상 노드 프로비저닝)
*	Ray 배포: KubeRay Operator
*	비용 추적: Kubecost (또는 AWS System Manager + Prometheus) + Ray Logical Resource tagging
*	테넌트 격리: Namespace 분리 및 Ray Cluster per Customer (or Namespace)
