# GitOps con Helm y Argo CD

Helm Chart de un servicio web simple (nginx) desplegado en un clúster local de Kubernetes (Kind) y sincronizado automáticamente por Argo CD a partir de este repositorio.

## Estructura

```
webapp/                    Helm Chart
  Chart.yaml
  values.yaml
  templates/
    _helpers.tpl
    configmap.yaml         página index.html servida por nginx
    deployment.yaml
    service.yaml           NodePort 30080
argocd/
  values-argocd.yaml       valores usados para instalar Argo CD con Helm
  application.yaml         Argo CD Application (sync automático)
kind-config.yaml           definición del clúster Kind
evidencias/                salidas de los comandos de validación
```

## Pasos realizados

### 1. Clúster local

```bash
kind create cluster --config kind-config.yaml --wait 180s
kubectl get nodes
```

El clúster publica dos puertos en el host: `8081` (webapp) y `8443` (UI de Argo CD).

### 2. Probar el chart

```bash
helm lint webapp
helm install prueba webapp -n prueba --create-namespace --wait
kubectl get deploy,svc,cm -n prueba
helm uninstall prueba -n prueba
```

### 3. Instalar Argo CD

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd -n argocd --create-namespace -f argocd/values-argocd.yaml --wait
```

UI: https://localhost:8443 (usuario `admin`). La contraseña inicial se obtiene con:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"
```

(el valor está en base64).

### 4. Application de Argo CD

```bash
kubectl apply -f argocd/application.yaml
kubectl get applications -n argocd
```

La Application apunta a la carpeta `webapp/` de la rama `main` de este repositorio y despliega en el namespace `webapp` con `syncPolicy.automated` (`prune` y `selfHeal`).

### 5. Cambio, sincronización y rollback

```bash
# cambio: editar webapp/values.yaml (replicaCount, image.tag, web.message)
git commit -am "Escalar a 4 replicas y actualizar nginx"
git push

kubectl get deploy,svc -n webapp        # Argo CD aplica el cambio solo

# rollback desde Git
git revert HEAD --no-edit
git push
```

## Flujo GitOps logrado

1. **Git es la única fuente de verdad.** El estado deseado de la aplicación (chart + `values.yaml`) vive en este repositorio; nadie ejecuta `kubectl apply` ni `helm upgrade` a mano contra el clúster.
2. **Argo CD observa el repositorio.** Cada 60 segundos consulta la rama `main`, renderiza el Helm Chart y compara el resultado con lo que realmente hay en el clúster.
3. **Si hay diferencia, sincroniza.** Cuando se hace `git push` de un cambio en `values.yaml`, la Application pasa a `OutOfSync` y, por tener sync automático, Argo CD aplica los manifiestos hasta volver a `Synced / Healthy`.
4. **Autocorrección.** Con `selfHeal`, un cambio manual en el clúster (por ejemplo `kubectl scale`) se revierte al valor definido en Git; con `prune`, los recursos eliminados del chart se eliminan del clúster.
5. **Rollback = `git revert`.** Para volver atrás se revierte el commit; Argo CD detecta el nuevo commit y devuelve el clúster al estado anterior. Todo cambio queda auditado en el historial de Git.
