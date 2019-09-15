## Deployment与Statefulset

statefulset其实本质是通过headless service进行访问的，创建一个无cluster ip的service，此时会通过service的labelselector创建每个pod对应的dns解析，<pod-name>.<svc-name>.cluster.local.svc



statefulset的spec中会定义seviceName选择对应的headless service