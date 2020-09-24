# VPN Site To Site ou Site To Cliente com WIREGUARD

No tutorial abaixo estarei exemplificando uma VPN de alta qualidade e com segurança diferentes dos demais tutoriais encontrados na internet que não nos detalham.

No tutorial abaixo usaremos dois linux ubuntu, porém pode ser aplicado em praticamente todas as variações de linux, verificamos isso no site da wireguard: '''https://www.wireguard.com/install/ '''

Também podemos realizar os links de vpns entre os equipamentos da mikrotik (RouterOS na versão 7.1 Beta development) e/ou Linux e RouterOS e também possivelmente no pFsense (Não verifiquei), vice e versa.

No cenário abaixo estarei utilizando como cenário a AWS, porém não há diferenças em realizar em on primesses, apenas terá alguns detalhes a mais para a AWS.

## Configuração

Observações: no caso estarei utilizando a AWS, realizei a criação de uma instância EC2 com o security grupo com o nome VPN-WIREGUARD(Apenas para detalhar) com a porta 22 do SSH para o meu IP público.

Após termos acessos via SSH o lado da Matriz da VPN utilizando ubuntu inciaremos instalando o repositório do wireguard com os comandos abaixo:

```
sudo add-apt-repository ppa:wireguard/wireguard
```

e em seguida necessário atualizarmos o repositório com o comando abaixo:

```
sudo apt update
```
agora precisamos instalar o wireguard:

```
sudo apt install wireguard-dkms wireguard-tools
```

Após a instalação iremos gerar as chaves publica e privada para a autenticação, pois o wireguard utiliza autenticação de ambos os lados:

```
wg genkey | tee chave_privada_key | wg pubkey > chave_publica_key

```

Utilizaremos o comando cat para mostrar os conteudos dos arquivos:

```
cat chave_privada_key
sCOoUpIdnuNd59oGrIqlL8w430alnJcH8ZsxVWccamo=
--------------------------------------------------------
cat chave_publica_key
aOMFtPc9XITALo+S7P05QYIX117xKqyAWTl6bMtOXnA=
```
por boa prática anotaremos em um bloco de notas temporariamente e identificando qual é pública e qual é a privada para evitar erros.

Em seguida criaremos o arquivo de configuração da vpn matriz:

```
vim /etc/wireguard/vpn-matriz01.conf
```
e em seguida começaremos as configurações com o conteúdo:

```
[Interface]
Address = 20.200.200.1/24
SaveConfig = true
PrivateKey = sCOoUpIdnuNd59oGrIqlL8w430alnJcH8ZsxVWccamo=
ListenPort = 51820
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = K2PrwhLtJ9zW8l2UT6FuI3A/FUf+sBPcnbcqJhgd+XQ=
AllowedIPs = 25.25.0.0/16, 20.200.200.2/32
```
```
[IMPORTANTE]

No campo PrivateKey colocaremos a chave privada gerada no servidor, e no campo PublicKey colocaremos a chave pública da filial/cliente!
```
```
[IMPORTANTE]
em AllowedIPs ele é utilizado como rota, muito tutoriais por aí ensinam de maneira que utilize a rota 0.0.0.0/0.
Em alguns casos isso causa falta de internet ou o uso da internet do lado da matriz.
Então configuraremos da seguinte maneira para que não utilizemos a internet da AWS.

AllowedIPs = 25.25.0.0/16, 20.200.200.2/32

Note que: 25.25.0.0/16 é minha rede local.

Note que: 20.200.200.2/32 é o ip do túnel do lado do cliente/filial
```

Deixaremos vazio o campo PublicKey por motivos de não termos gerado a chave publica ainda do lado da filial/client:

após criar o conteúdo e salvar o arquivo, precisamos deixar o arquivo seguro:

```
sudo chown root:root /etc/wireguard/vpn-matriz01.conf
sudo chmod 600 /etc/wireguard/vpn-matriz01.conf
```
como toda conexão da VPN passará por dentro da vpn da matriz precisaremos habilitar o FORWARD:

```
sudo vim /etc/sysctl.conf

procure a linha:
#net.ipv4.ip_forward=1

e mude para:
net.ipv4.ip_forward=1
```
logo em seguida utilizaremos o comando para atualizar:
```
sudo sysctl -p
```
podemos utilizar também para garantir:

```
echo 1 > /proc/sys/net/ipv4/ip_forward
```

Após esse processo passaremos para o lado do cliente:
Realizaremos os mesmo procedimentos de instalação do wireguard:

```
sudo add-apt-repository ppa:wireguard/wireguard
sudo apt-get update
sudo apt-get install wireguard-dkms wireguard-tools
```

E criar a chave publica e privada:
```
wg genkey | tee cliente_private_key | wg pubkey > cliente_publica_key

cat cliente_private_key
WHB4DvUqM9ugW0SWx7U9QGjKiNLc1o/JFY/+3lv3f1I=
-----------------------------------------------------
cat cliente_publica_key
K2PrwhLtJ9zW8l2UT6FuI3A/FUf+sBPcnbcqJhgd+XQ=

```
Com a chave Publica em mãos voltaremos para o servidor Matriz e colocaremos a chave publica:

```
PublicKey = K2PrwhLtJ9zW8l2UT6FuI3A/FUf+sBPcnbcqJhgd+XQ=
```

voltando para as configurações do cliente e com a matriz devidamente configurada, criaremos o arquivo de configuração do cliente:
```
sudo vim /etc/wireguard/client01.conf
```
```
[Interface]
Address = 20.200.200.2/32
PrivateKey = WHB4DvUqM9ugW0SWx7U9QGjKiNLc1o/JFY/+3lv3f1I=

[Peer]
PublicKey = aOMFtPc9XITALo+S7P05QYIX117xKqyAWTl6bMtOXnA=
Endpoint = 34.228.16.47:51820
AllowedIPs = 172.31.0.0/16, 20.200.200.1/32
PersistentKeepalive = 21
```
Permissões de segurança no arquivo (caso o usuario não tenha permisão do sudo/root, sera necessario modificar de acordo com o usuario):

```
sudo chown root:root /etc/wireguard/cliente01.conf
sudo chmod 600 /etc/wireguard/client01.conf
```

Note que no campo PublicKey colocamos a chave Publica da MATRIZ
```
[IMPORTANTE]

Endpoint é o local onde colocaremos o IP fixo publico da instância da AWS!

Endpoint = 34.228.16.47:51820 ip público instância e porta

AllowedIPs = 172.31.0.0/16, 20.200.200.1/32

Note que: 172.31.0.0/16 é a range de ip da minha VPC da AWS

Note que: 20.200.200.1/32 é o IP do túnel da matriz.

[LIBERAR PORTA]
No Security Group será necessário liberar a porta:

51820 UDP
e também:

ALL TRAFFIC do Security Group "Default" para o
Security Group da instância, nesse caso o que criei no início do tutorial "VPN-WIREGUARD"
```

## Finalizando
Testando a inicialização da VPN da Matriz:

```
wg-quick up matriz01
```

Testando a inicialização da VPN do Client:
```
wg-quick up client01
```

Habilitando no boot a inicialização da VPN na MATRIZ:

```
systemctl enable wg-quick@matriz01.service
```

Habilitando no boot a inicialização da VPN do cliente:
```
systemctl enable wg-quick@client01.service
```
## Testando

Do lado da Matriz:
```
ping 20.200.200.2
ping no ip local da sua (rede) exemplo: 25.25.25.23 que é o ip da maquina cliente no meu caso.
```

Do lado do cliente:
```
ping 20.200.200.1
ping no ip local das instâncias exemplo: 172.31.1.200 no meu caso.
```
## Contributing
Pull requests are welcome. For major changes, please open an issue first to discuss what you would like to change.

Please make sure to update tests as appropriate.

## License
[MIT](https://choosealicense.com/licenses/mit/)
