# DevOps-Lab

Notas técnicas propias de las tecnologías que voy tocando en el día a día como DevOps.
Todo en markdown, organizado por temas. Es un cuaderno de trabajo, no documentación oficial.

## Índice

Cada carpeta tiene su propio README con el orden de lectura recomendado y qué cubre cada
nota: se abre pulsando en el título de la sección.

### [Kubernetes](kubernetes/README.md)
- [Arquitectura](kubernetes/01-arquitectura.md)
- [kubeconfig y kubectl](kubernetes/02-kubeconfig-y-kubectl.md)
- [Pods](kubernetes/03-pods.md)
- [Deployment y ReplicaSet](kubernetes/04-deployments-y-replicasets.md)
- [Service e Ingress](kubernetes/05-services-e-ingress.md)
- [Networking (1): LoadBalancer y MetalLB](kubernetes/06-networking-loadbalancer-metallb.md)
- [Networking (2): Ingress Controllers y cert-manager](kubernetes/07-ingress-controllers-y-cert-manager.md)
- [Almacenamiento: PV, PVC y Longhorn](kubernetes/08-almacenamiento-longhorn.md)
- [RBAC](kubernetes/09-rbac.md)
- [K9s](kubernetes/10-k9s.md)
- [Kinds abreviados (chuleta)](kubernetes/11-kinds-abreviados.md)
- [Networking (3): resumen con CNI y flujo completo](kubernetes/12-networking-resumen-cni.md)
- [Entorno local y productividad con kubectl](kubernetes/13-kubectl-productividad-y-entorno-local.md)
- [Helm y Operators](kubernetes/14-helm-y-operators.md)

### [Ansible](ansible/README.md)
- [Conceptos básicos](ansible/01-conceptos-basicos.md)
- [Ansible en Linux vs Windows](ansible/02-linux-vs-windows.md)
- [Variables: vars vs defaults](ansible/03-variables-vars-defaults.md)
- [Módulos frecuentes](ansible/04-modulos-frecuentes.md)
- [Jinja2](ansible/05-jinja2.md)
- [Galaxy privado y colecciones](ansible/06-galaxy-y-colecciones.md)
- [Action plugins](ansible/07-action-plugins.md)
- [YAML, gather facts e idempotencia](ansible/08-yaml-e-idempotencia.md)
- [Directivas y conceptos clave](ansible/09-directivas-y-conceptos.md)
- [AWX (Automation Controller)](ansible/10-awx.md)

### [Docker](docker/README.md)
- [Fundamentos e imágenes (Dockerfile)](docker/01-fundamentos-y-dockerfile.md)
- [Compose y Docker Engine](docker/02-compose-y-engine.md)

### [Seguridad](seguridad/README.md) (SSH / TLS / cifrado)
- [SSH: fundamentos y sshd_config](seguridad/01-ssh-fundamentos.md)
- [SCP y SFTP](seguridad/02-scp-y-sftp.md)
- [Clientes Windows y agent forwarding](seguridad/03-clientes-windows-y-agent-forwarding.md)
- [MFA en SSH](seguridad/04-mfa-ssh.md)
- [SSHFS y VS Code Remote](seguridad/05-sshfs-y-vscode-remote.md)
- [Túneles SSH: X11, port forwarding y SOCKS](seguridad/06-tuneles-ssh-x11-socks.md)
- [TLS, SSL y PKI](seguridad/07-tls-ssl-pki.md)
- [CA y gestión de certificados](seguridad/08-ca-y-certificados.md)
- [AES-256](seguridad/09-aes-256.md)
- [Ataques históricos a SSL/TLS](seguridad/10-ataques-tls.md)

### [Vault / OpenBao](vault/README.md)
- [Vault: motor KV](vault/01-vault-kv.md)
- [OpenBao: barrier, seal e init](vault/02-openbao-seal-e-init.md)
- [OpenBao: HA y Raft](vault/03-openbao-ha-y-raft.md)

### [PowerShell](powershell/README.md)
- [Alias y equivalencias con Linux/CMD](powershell/01-alias-y-equivalencias.md)
- [Navegación, pilas y pipeline](powershell/02-navegacion-y-pipeline.md)
- [PSDrive y proveedores](powershell/03-psdrive-y-proveedores.md)
- [Splatting](powershell/04-splatting.md)
- [WinRM y remoting](powershell/05-winrm-y-remoting.md)

### [Windows](windows/README.md)
- [Atajos de teclado](windows/01-atajos-teclado.md)
- [Acceso remoto: RDP y recursos compartidos](windows/02-acceso-remoto.md)
- [Instalar una app Java como servicio con Tomcat](windows/03-app-java-como-servicio.md)
- [DLLs](windows/04-dlls.md)

### [Redes](redes/README.md)
- [Puertos y protocolos (TCP/UDP)](redes/01-puertos-y-protocolos.md)
- [Puertos más comunes](redes/02-puertos-comunes.md)
- [Transferencia de archivos y repositorios de artefactos](redes/03-transferencia-y-artefactos.md)
- [sslip.io](redes/04-sslip-io.md)
- [Nmap y Telnet](redes/05-nmap-y-telnet.md)

### [Shell / Linux](linux-shell/README.md)
- [Shell, zsh y personalización](linux-shell/01-shell-y-zsh.md)
- [Atajos de consola](linux-shell/02-atajos-consola.md)
- [GRUB](linux-shell/03-grub.md)
- [Permisos](linux-shell/04-permisos.md)
- [sed](linux-shell/05-sed.md)

### [Observabilidad](observabilidad/README.md)
- [Prometheus, Node Exporter y Grafana](observabilidad/prometheus-y-grafana.md)

### [Git](git/README.md)
- [Git Flow](git/gitflow.md)

### [CI/CD](ci-cd/README.md)
- [CI/CD con GitLab](ci-cd/01-gitlab-ci.md)

### Lenguajes: [Python](python/README.md) y [Go](go/README.md)
- [Python: pandas](python/pandas.md)
- [Go (Golang)](go/fundamentos.md)

### [Fundamentos](fundamentos/README.md)
- [Parsers y codificación de texto (UTF-8 / BOM)](fundamentos/parsers-y-codificacion.md)
- [Expresiones regulares (regex)](fundamentos/regex-basico.md)
- [JSON](fundamentos/json.md)
- [Base64](fundamentos/base64.md)

### [Herramientas](herramientas/README.md)
- [Molecule](herramientas/molecule.md)
- [JAR (Java Archive)](herramientas/jar.md)
- [Wireshark](herramientas/wireshark.md)
- [Claude Code (CLI)](herramientas/claude-code.md)
