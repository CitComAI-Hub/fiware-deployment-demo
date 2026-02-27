# Desplegar Lepus en el provider

## Contexto
Lepus es un componente que actúa como un traductor entre NGSIv2 y NGSI-LD. Es decir, recibe peticiones NGSIv2 y contesta con NGSI-LD. Es un servidor HTTP hecho con Express (Node.js). Con la configuración actual del provider, APISIX apunta a Lepus (Ver línea https://github.com/CitComAI-Hub/fiware-deployment-demo/blob/5b516433addd242f38351147405bbdc58398990a/provider/values.yaml#L107).

<div style="text-align: center;">
    <img src="mvds_valencia.png" alt="Arch" width="900">
</div>

<br><br>
- El Lepus que desplegamos es un **fork** nuestro basado en el original.
    - Repo original: https://github.com/jason-fox/lepus 
    - Fork: https://github.com/CitComAI-Hub/lepus. 
        - **Cambios que aplicamos:** 
            
            - Autenticación con la plataforma del Ayuntamiento de Valencia: Es decir, cada vez que Lepus recibe una petición NGSIv2, se autentica con la plataforma del Ayuntamiento de Valencia para obtener un token de acceso (en caso de que haya caducado), y luego utiliza ese token para hacer las peticiones a la plataforma del Ayuntamiento de Valencia. Para configurar la autenticación, se han añadido varias variables de entorno (explicado más abajo).
            
            - Contexto dinámico (en el original es fijo): Es decir, dada una petición NGSIv2, se lee el tipo de entidad y se adjunta el `@context` correspondiente en la respuesta NGSI-LD. En el original el contexto es fijo y se fija mediante una variable de entorno.

            - Asignación automática de tenant y servicePath: Eliminamos la necesidad de enviar los headers fiware-service y fiware-servicepath en las peticiones. Ahora Lepus asigna el tenant ("tef_vlci") y el servicePath de forma dinámica según el tipo de entidad recibida, usando un mapa estático interno. Para entidades como `AirQualityObserved`, que pueden tener varios ServicePath, Lepus selecciona automáticamente el primero del array. Si se requiere una lógica más avanzada para elegir el ServicePath, se puede adaptar fácilmente. Para actualizar el mapa de tipos a ServicePath, basta con modificar el archivo `lib/entityServicePaths.js`.

- Para desplegar Lepus, se ha creado un pod de Kubernetes ([`lepus-dev-pod.yaml`](https://github.com/CitComAI-Hub/fiware-deployment-demo/blob/main/lepus-dev-pod.yaml)) que ejecuta el código de Lepus directamente. Esto es útil para desarrollo, pero no es una forma óptima de desplegarlo en producción. En producción, lo ideal sería dockerizar Lepus y desplegarlo como un contenedor dentro de Kubernetes. Está en la lista de tareas pendientes.

- Las variables de entorno definidas en `lepus-dev-pod.yaml` son bastante autoexplicativas, pero destacar:
    - `AUTH_URL`: URL para autenticarse con la plataforma del Ayuntamiento de Valencia. Es la URL que se utiliza para obtener un token de acceso. 
    - `AUTH_USER` y `AUTH_PASSWORD`: Credenciales para autenticarse con la plataforma del Ayuntamiento de Valencia. Estas credenciales se han proporcionado desde el Ayuntamiento de Valencia para que podamos acceder a su plataforma. El usuario está en texto plano, pero la contraseña se ha guardado como un secreto de Kubernetes (`lepus-auth-secret`) para no exponerla directamente en el código. En `lepus-dev-pod.yaml`, se hace referencia a este secreto para obtener la contraseña. Quizás en el futuro sería mejor tener ambas credenciales (usuario y contraseña) en un secreto.
    - `NGSI_V2_TIMEOUT=10000`: ***Maximum length of time to access the NGSI-v2 Orion Context Broker URL in milliseconds***. Le pedí a Stefan que aumentara el `timeout` por defecto porque en ocasiones la plataforma del Ayuntamiento de Valencia responde un poco lento y se quedaba pillado esperando la respuesta. Añadió esta variable de entorno para poder configurarlo. Puede ser necesario ajustarlo dependiendo de la velocidad de respuesta de la plataforma del Ayuntamiento de Valencia. Para testing, lo he puesto a 10 segundos (10000 ms), pero en producción lo ideal sería tener un timeout más bajo. 
    - `NODE_TLS_REJECT_UNAUTHORIZED=0`: Desactivada comprobación SSL. Motivo: The SSL certificate of the server you are trying to connect to is not valid for the hostname you are using. Specifically, the error says: ***Hostname/IP does not match certificate's altnames: Host: cb.vlci.valencia.es. is not in the cert's altnames: DNS:*.iotplatform.telefonica.com***. Why does this happen? El servidor al que te estás conectando (cb.vlci.valencia.es) está usando un certificado SSL que no coincide con ese dominio. Por seguridad, Node.js (y la mayoría de los clientes HTTPS) rechazan conexiones a servidores cuyo certificado no coincide con el hostname. `NODE_TLS_REJECT_UNAUTHORIZED=0` indica a Node.js que ignore los errores de validación del certificado SSL. Es útil para testing, pero no seguro para producción. Queda pendiente investigar una solución más segura para producción, como por ejemplo configurar Lepus para que confíe en el certificado específico que está usando la plataforma del Ayuntamiento de Valencia (porque dudo que ellos vayan a cambiar su certificado).
    - Lepus tiene otras variables de entorno (opcionales). Ver https://github.com/CitComAI-Hub/lepus?tab=readme-ov-file#configuration 

## Despliegue

https://github.com/CitComAI-Hub/fiware-deployment-demo/blob/main/lepus-dev-pod.yaml

```bash
# Asegúrate de tener el kubeconfig del provider configurado
export KUBECONFIG=k3s-provider.yaml

# Crear el secreto con la contraseña de autenticación
kubectl create secret generic lepus-auth-secret \
    --from-literal=AUTH_PASSWORD=XXXXXXXXX \
    -n provider

# Verificar que el secreto se ha creado correctamente
kubectl get secret lepus-auth-secret -n provider

# Desplegar el pod de Lepus        
kubectl apply -f lepus-dev-pod.yaml
```

## Uso

### Bash
```bash
export ACCESS_TOKEN="your_access_token_here"

curl -s -X GET 'http:///mp-data-service.54.155.89.222.nip.io/ngsi-ld/v1/entities?type=AirQualityObserved' \
  --header 'Accept: application/ld+json' \
  --header "Authorization: Bearer ${ACCESS_TOKEN}" 
  
curl -s -X GET 'http:///mp-data-service.54.155.89.222.nip.io/ngsi-ld/v1/entities?type=WasteContainer' \
  --header 'Accept: application/ld+json' \
  --header "Authorization: Bearer ${ACCESS_TOKEN}" 
```

### Python
```python
def get_all_WasteContainers():
    # Get OID4VC/VP access token
    keycloak_url = os.environ.get("KEYCLOAK_URL", "http://keycloak-consumer.63.33.94.64.nip.io")
    credential_configuration_id = os.environ.get("CREDENTIAL_CONFIGURATION_ID", "operator-credential")
    mp_data_service_url = os.environ.get("MP_DATA_SERVICE_URL", "http://mp-data-service.54.155.89.222.nip.io")
    
    access_token = get_access_token_oid4vp(keycloak_url, credential_configuration_id)

    # Data endpoint
    url = f"{mp_data_service_url}/ngsi-ld/v1/entities?type=WasteContainer"
    headers = {
        'Accept': 'application/ld+json',
        'Authorization': f'Bearer {access_token}'
    }
    response = requests.get(url, headers=headers)
    response.raise_for_status()
    return response.json()
```
Código extraído de https://github.com/CitComAI-Hub/waste-collection-demo/blob/mvds/server.py