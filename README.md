Vamos revisar seu cenário, identificar possíveis problemas e te orientar a **recriar do zero** o stack de monitoramento (Prometheus + Grafana) usando ArgoCD e Helm.

---

## 1. **Análise dos Problemas**

Pelos pods, há **conflito de releases** e pods com erro:

- Você tem pods com prefixo `monitoring-stack-` (do Helm/ArgoCD) e outros com prefixo `prometheus-` (provavelmente de uma instalação anterior, talvez `kubectl apply` ou outro Helm release).
- O pod `prometheus-monitoring-stack-kube-prom-prometheus-0` está com `Init:Error`.
- O pod `monitoring-stack-prometheus-node-exporter-xrj6w` está `Pending`.
- O pod `alertmanager-monitoring-stack-kube-prom-alertmanager-0` está `Terminating`.

**Causa provável:**
- Recursos duplicados no mesmo namespace (`monitoring`), conflitos de CRDs, ou sobras de releases antigas.
- Possível erro de configuração no Helm values.

---

## 2. **Procedimento Seguro para RECRIAR do ZERO**

### **A. Limpeza do Namespace**

. **Remova a aplicação do ArgoCD:**
   ```sh
   argocd app delete monitoring-stack --cascade
   ```
Ou via UI do ArgoCD.

. **Remova releases antigas do Helm (se existirem):**
   ```sh
   helm list -n monitoring
   helm uninstall <release-name> -n monitoring
   ```

. **Remova recursos órfãos:**
   ```sh
   kubectl delete all --all -n monitoring
   kubectl delete pvc --all -n monitoring
   kubectl delete configmap --all -n monitoring
   kubectl delete secret --all -n monitoring
   ```

. **(Opcional, mas recomendado)**  
Se não houver mais nada importante no namespace:
   ```sh
   kubectl delete ns monitoring
   kubectl create ns monitoring
   ```

---

### **B. Revise o YAML do ArgoCD**

Seu YAML está **quase correto**, mas recomendo alguns ajustes para evitar problemas comuns:

- **`skipCrds: true`**: só use se os CRDs já existem e estão atualizados. Se for a primeira instalação, remova ou defina como `false`.
- **`serviceMonitorSelector`**: está correto, mas garanta que seus ServiceMonitors tenham o label `release: monitoring-stack`.
- **`adminPassword`**: para produção, use um Secret, não senha em texto.
- **`NodePort`**: ok para dev, mas para produção prefira Ingress.

#### **YAML REVISADO:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: monitoring-stack
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://prometheus-community.github.io/helm-charts
    chart: kube-prometheus-stack
    targetRevision: 65.1.0
    helm:
      skipCrds: false # Instale os CRDs se for a primeira vez
      values: |
        prometheus:
          prometheusSpec:
            serviceMonitorSelectorNilUsesHelmValues: false
            podMonitorSelectorNilUsesHelmValues: false
            ruleSelectorNilUsesHelmValues: false
            serviceMonitorNamespaceSelector: {}
            podMonitorNamespaceSelector: {}
            probeNamespaceSelector: {}
            serviceMonitorSelector:
              matchLabels:
                release: monitoring-stack
        grafana:
          adminPassword: admin
          service:
            type: NodePort
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

---

### **C. Reaplique a aplicação**

. **Adicione o YAML revisado ao seu repositório Git.**
. **No ArgoCD, sincronize a aplicação:**
- Via UI: clique em "Sync".
- Ou via CLI:
  ```sh
  argocd app create -f <seu-arquivo-yaml>
  argocd app sync monitoring-stack
  ```

---

### **D. Verifique os pods**

```sh
kubectl get pods -n monitoring
```
- Todos os pods devem ficar em `Running` (pode demorar alguns minutos na primeira vez).

---

### **E. Dicas Finais**

- Se o Prometheus continuar com erro, rode:
  ```sh
  kubectl describe pod <nome-do-pod> -n monitoring
  kubectl logs <nome-do-pod> -n monitoring
  ```
  e compartilhe o erro para análise detalhada.
- **Evite múltiplas releases** do Prometheus/Grafana no mesmo namespace.
- **ServiceMonitor**: garanta que o label `release: monitoring-stack` está presente nos seus ServiceMonitors customizados.

---

## **Resumo dos Passos**

. **Remova tudo do namespace `monitoring`** (inclusive releases antigas).
. **Aplique o YAML revisado** via ArgoCD.
. **Aguarde os pods subirem**.
. **Verifique os logs** se houver erro.

Se seguir esses passos, seu stack de monitoramento será recriado do zero, limpo e funcional! Se algum pod continuar com erro, envie o log para análise.
/@stackspotai {"sources":[{"type":"project_file","name":"repos/java-k8s-manifests","document_score":100.0,"path":"monitoring\\argocd-application-prometheus.yaml","slug":"ffb55211-c562-45ef-a103-bdc21c560e9b-reposjava-k8s-manifests","document_id":"01K6Y4AJDRJA5RGDQRZVWDNKX8"},{"type":"project_file","name":"repos/java-k8s-manifests","document_score":100.0,"path":"monitoring\\serviceMonitor-java-k8s-app.yaml","slug":"ffb55211-c562-45ef-a103-bdc21c560e9b-reposjava-k8s-manifests","document_id":"01K6V73N9SYA49YQAF7Z4XS0EG"},{"type":"project_file","name":"repos/java-k8s-manifests","document_score":100.0,"path":"my-app\\argocd-application.yaml","slug":"ffb55211-c562-45ef-a103-bdc21c560e9b-reposjava-k8s-manifests","document_id":"01K6V73KTEEVQ8JH4ZN6HPB7G3"},{"type":"project_file","name":"repos/java-k8s-manifests","document_score":100.0,"path":"monitoring\\prometheus-ingress.yaml","slug":"ffb55211-c562-45ef-a103-bdc21c560e9b-reposjava-k8s-manifests","document_id":"01K6RPMDH3M3SR6H2TE73EPXA5"},{"type":"project_file","name":"repos/java-k8s-manifests","document_score":100.0,"path":"README.md","slug":"ffb55211-c562-45ef-a103-bdc21c560e9b-reposjava-k8s-manifests","document_id":"01K61T412RAS5GH7M9FHWGYD7Y"},{"type":"project_file","name":"repos/java-k8s-manifests","document_score":100.0,"path":"my-app\\deployment.yaml","slug":"ffb55211-c562-45ef-a103-bdc21c560e9b-reposjava-k8s-manifests","document_id":"01K6V73S3F8XDCFP8JXTQ2D19X"},{"type":"project_file","name":"repos/java-k8s-manifests","document_score":100.0,"path":"my-app\\service.yaml","slug":"ffb55211-c562-45ef-a103-bdc21c560e9b-reposjava-k8s-manifests","document_id":"01K6V73QFFA8MBGJFC3B7XSE7W"}]} /

/@stackspotai {"messageId":"01K6Z4MAJVH551H3Q07SDWVTKA"} /
