# 监控栈配置（GitOps 版本管理）
- vmstack-values.yaml / vlogs-values.yaml：victoria-metrics-k8s-stack + victoria-logs-single 的 helm values
- 密钥全部经 SealedSecret 注入（见 manifests/monitoring/sealed-*.yaml）：
  - grafana: admin.existingSecret=grafana-admin
  - alertmanager: telegram bot_token_file=/etc/tg/token（挂载 tg-bot-token secret）
- 部署：helm upgrade --install vmstack vm/victoria-metrics-k8s-stack -n monitoring -f vmstack-values.yaml
- SealedSecret 由 ArgoCD（apps/monitoring-secrets）同步，sealed-secrets controller 解密
