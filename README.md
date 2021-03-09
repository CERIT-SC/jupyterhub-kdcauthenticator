# KDCAuthenticator

Forked version of original KDCAuthenticator. The change lies in function `authenticate(self, handler, data)` where already existing function `kerberos.checkPassword` is used instead of `authGSSServerStep`. From provided `data`, username is returned (username@REALM split on `@`, the first part returned).

KDC authenticator allows to authenticate the JuypterHub user using Kerberos protocol.

# Install, Configure and Run

This is an original tutorial from base repo.

1. Install KDC Authenticator -

    Run the following command at kdcauthenticator directory

    ```
    pip3 install jupyterhub-kdcauthenticator
    ```

    Or clone the repository and install -
    ```
    git clone https://github.com/bloomberg/kdcauthenticator.git
    cd kdcauthenticator
    pip3 install -e .
    ```

2. Configure JupyterHub for KDC Authenticator

    Add the following line to the jupyterHub config file
    ```
    c.JupyterHub.authenticator_class = 'kdcauthenticator.kdcauthenticator.KDCAuthenticator'
    ```
    Optionally you can add the following lines to create local system users
    ```
    c.LocalAuthenticator.add_user_cmd = ['adduser', '-m']
    c.LocalAuthenticator.create_system_users = True
    ```

3. The Service principal for JupyterHub authenticator is configured to "HTTP" but can be configured by -

    ```
    c.KDCAuthenticator.service_name = '<HTTP-Service-Principal>'
    ```

4. Run the JupyterHub command with Kerberos environment variables -

    ```
    KRB5_CONFIG=[Kerberos-config-path] KRB5_KTNAME=[HTTP-Service-Principle-Keytab-path] jupyterhub --ip=0.0.0.0 --port=8000 --no-ssl --config=[jupyterHub-config-file-path]
    ```


# Use in zero2jupyterhub

1. Create image 

Create new docker image from `jupyterhub/k8s-hub:tag` and change user to `root`. Install needed dependencies:
 - `libkrb5-dev` and `git-all` (to clone this repo)

Clone the repo and chown the directory where repo is located to user `jovyan` (default jupyter user). Change back to user `jovyan` and install with `pip` kerberos and kdcauthenticator. 

Our dockerhub with modified image: [cerit dockerhub](https://hub.docker.com/r/cerit/jupyterhub/tags?page=1&ordering=last_updated)

2. Z2JH config

Kerberos needs 2 files to function: 
 - krb5.conf
 - krb5.keytab

The keytab file is sensitive and shouldn't be stored publicly. Therefore resource type `Secret` is used. 

Create as:
```
kubectl create secret generic krb-secret --from-file [keytab file] -n [namespace]
```

Conf file can be create as resource type `ConfigMap` using command:
```
kubectl create configmap krb-cm --from-file [conf file] -n [namespace]
```

The hub section in `values.yaml` in z2jh chart has to be modified. `ExtraVolumes` and `extraVolumeMounts` mount created secret and configmap into the hub pod. 
Specify particular image name and tag with kdcautheticator installed, otherwise default z2jh image is used. `ExtraEnv` together with `extraConfig` are related to kdcauthenticator. Don't forget to change `service_name` in `c.KDCAuthenticator.service_name` and set realm name in `KERBEROS_REALM` environment variable. Default is None which leads to error and throwing an exception.

```
hub:
  image:                                                                        
    name: cerit/jupyterhub                                                      
    tag: [tag]
  extraEnv:
    - name: KRB5_CONFIG                                                         
      value: /etc/krb5.conf # (location specified in extraVolumeMounts)
    - name: KRB5_KTNAME                                                         
      value: /etc/krb5.keytab # (location specified in extraVolumeMounts)
    - name: KERBEROS_REALM
      value: [default realm name] # (default None) 
  extraVolumes:                                                                 
      - name: krb-conf                                                          
        configMap:                                                              
          name: krb-cm                                                          
      - name: keytab                                                            
        secret:                                                                 
          secretName: krb-secret                                                
  extraVolumeMounts:                                                            
    - name: krb-conf                                                            
      mountPath: "/etc/krb5.conf" #(or other)                                  
      subPath: "krb5.conf"                                                      
    - name: keytab                                                              
      mountPath: "/etc/krb5.keytab"  #(or other)
      subPath: "krb5.keytab" 
  extraConfig:
    krb-extra: | 
      from kdcauthenticator.kdcauthenticator import KDCAuthenticator            
        c.JupyterHub.authenticator_class = KDCAuthenticator                       
        c.KDCAuthenticator.service_name = "http@[your_service_name]" 
```

3. Redeploy helm chart with command similar to ` helm upgrade --cleanup-on-fail --install jupyterhub jupyterhub/jupyterhub --namespace [namespace] --version=0.11.1 --values values.yaml``




