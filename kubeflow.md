Kubeflow is not a signle project. <br>
it is group of multiple projects which is used in ML life cycle. <br>
You can install any projects as per the requirements. <br>

**Different teams are working on different modules, thats why kubeflow is moduler.** <br>

### Kubeflow projects"

| Project            | Purpose                                    |
| ------------------ | ------------------------------------------ |
| Kubeflow Pipelines | ML workflow orchestration                  |
| KServe             | Model serving / inference                  |
| Katib              | Hyperparameter tuning                      |
| Kubeflow Notebooks | Jupyter notebook environments              |
| Training Operator  | Distributed training (PyTorch, TensorFlow) |
| Central Dashboard  | Kubeflow UI                                |
| Profiles           | Multi-user isolation                       |

without Kubeflow:
<img width="3330" height="1610" alt="ml-lifecycle-without-kubeflow" src="https://github.com/user-attachments/assets/05d7e08b-b62e-4c0c-892e-3bf88f30ad47" />

With Kubeflow:
<img width="3768" height="1683" alt="ml-lifecycle0with-kubeflow" src="https://github.com/user-attachments/assets/c0eac387-8e1a-40dc-be34-933d266f66fb" />


### Installing kubeflow pipeline:
```
kubectl apply -k github.com/kubeflow/pipelines/manifests/kustomize/cluster-scoped-resources
kubectl apply -k github.com/kubeflow/pipelines/manifests/kustomize/env/dev
```

