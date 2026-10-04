# Knote

Aplicación Spring Boot para publicar notas y subir imágenes. MongoDB conserva las
notas y MinIO guarda las imágenes en un bucket persistente; Knote no escribe los
archivos subidos en el disco local de sus contenedores, por lo que se puede
escalar a varias réplicas.

## Ejecutar localmente con Docker Compose

Requisitos: Docker Desktop con el motor Linux activo.

```powershell
docker compose up --build
```

Abre <http://localhost:8080>. La consola de MinIO está en
<http://localhost:9001> (usuario y contraseña locales predeterminados:
`minioadmin`). Los datos se conservan en los volúmenes `mongo-data` y
`minio-data`.

Para detener la aplicación:

```powershell
docker compose down
```

## Desplegar en Minikube

Requisitos: Docker Desktop, Minikube, kubectl y Java 21/Maven si también se
ejecutará Spring Boot desde el host.

1. Inicia el clúster y verifica que el nodo esté listo:

   ```powershell
   minikube start --driver=docker
   minikube status
   kubectl get nodes
   ```

2. Construye la imagen directamente en el daemon de Minikube:

   ```powershell
   minikube image build -t knote:local .
   ```

3. Crea un Secret local para las credenciales de MinIO. Estos valores de
   ejemplo son únicamente para desarrollo; no reutilices contraseñas reales.

   ```powershell
   kubectl create secret generic knote-minio --from-literal=access-key=minioadmin --from-literal=secret-key=minioadmin
   ```

4. Despliega MongoDB, MinIO y Knote, espera a que los recursos estén listos y
   consulta los pods:

   ```powershell
   kubectl apply -f kube
   kubectl rollout status deployment/mongo
   kubectl rollout status deployment/minio
   kubectl rollout status deployment/knote
   kubectl get pods,pvc,services
   ```

   MinIO crea el bucket `image-storage` al iniciar la aplicación. Si MinIO aún
   no está disponible, Knote reintenta la inicialización cada cinco segundos.

5. Abre la URL local de Knote y comprueba que ambas réplicas responden:

   ```powershell
   minikube service knote --url
   kubectl get pods -l app=knote
   ```

6. Para probar persistencia, sube una imagen y publica una nota que la
   referencie; elimina un pod de Knote y verifica que la nota y la imagen sigan
   disponibles desde la URL de servicio.

7. Al finalizar:

   ```powershell
   kubectl delete -f kube/knote.yaml
   kubectl delete deployment mongo minio
   kubectl delete service mongo minio
   kubectl delete secret knote-minio
   minikube stop
   ```

   Los PVC se conservan para mantener los datos locales. Para borrarlos
   explícitamente junto con los datos persistidos, ejecuta
   `kubectl delete pvc mongo-pvc minio-pvc`.

## Publicar la imagen en Docker Hub

Reemplaza `<usuario>` por tu Docker ID y asegúrate de haber iniciado sesión con
`docker login`:

```powershell
docker build -t <usuario>/knote:1.0.0 .
docker push <usuario>/knote:1.0.0
```

Para desplegar la imagen publicada en lugar de la imagen local, cambia
`image: knote:local` en `kube/knote.yaml` por `<usuario>/knote:1.0.0`.

## Evidencias para la entrega

Guarda capturas reales en `docs/screenshots/` o adjúntalas a la entrega:

1. `minikube status` y `kubectl get nodes` mostrando el clúster local activo.
2. `kubectl get pods,pvc,services` con Knote, MongoDB y MinIO listos.
3. Navegador mostrando Knote dentro del clúster, con una nota e imagen cargada.
4. Navegador mostrando Knote local en `http://localhost:8080` con una imagen
   cargada usando MinIO.
5. (Opcional) Consola de MinIO mostrando el bucket `image-storage` y el objeto.

No se deben presentar capturas simuladas. Los enlaces a GitHub y Docker Hub
dependen de publicar los cambios en las cuentas correspondientes.
