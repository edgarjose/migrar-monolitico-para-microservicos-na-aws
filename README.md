# Migrar WordPress entre EC2 e RDS

Procedimento de backup e restauro dos ficheiros e da base de dados de um WordPress. O nome histórico do repositório refere microserviços, mas os passos aqui descritos **não decompõem a aplicação em microserviços**: transferem um site e separam a sua base de dados.

## Âmbito

```mermaid
flowchart LR
  Original[WordPress original] -->|Ficheiros e dump| S3[Bucket privado]
  S3 --> Nova[EC2 de destino]
  S3 -->|Importação SQL| RDS[RDS de destino]
  Nova --> RDS
```

O procedimento pressupõe uma EC2 de destino com Apache/PHP compatíveis com a versão do WordPress e uma base de dados RDS já criada. A criação desses recursos não é automatizada neste repositório.

## Antes de migrar

- Ensaiar numa cópia; planear a janela de escrita/manutenção para obter um backup consistente.
- Usar um bucket privado, permissões limitadas e encriptação para os backups.
- Registar versões de PHP, MySQL/MariaDB, WordPress, plugins e caminho real do site.
- Guardar `wp-config.php` como segredo: contém credenciais e não deve entrar no Git.
- Confirmar conectividade da EC2 para o RDS sem publicar a porta 3306 na internet.

## Backup na origem

Substituir os valores pelos teus, sem colocar a palavra-passe na linha de comando:

```bash
export BACKUP_BUCKET=SEU_BUCKET_PRIVADO
mysqldump -h HOST_ORIGEM -u UTILIZADOR_ORIGEM -p --single-transaction wordpressdb > wordpressdb.sql
sudo tar -C /var/www -czf wordpress.tar.gz html
aws s3 cp wordpressdb.sql "s3://$BACKUP_BUCKET/wordpress-migration/wordpressdb.sql"
aws s3 cp wordpress.tar.gz "s3://$BACKUP_BUCKET/wordpress-migration/wordpress.tar.gz"
```

`--single-transaction` pressupõe tabelas transacionais e não torna os ficheiros do site consistentes com alterações simultâneas. Não usar estes comandos como substituto de uma estratégia de backup testada.

## Restauro no destino

Num diretório de trabalho protegido na EC2 de destino:

```bash
aws s3 cp "s3://$BACKUP_BUCKET/wordpress-migration/wordpressdb.sql" .
aws s3 cp "s3://$BACKUP_BUCKET/wordpress-migration/wordpress.tar.gz" .
mysql -h ENDPOINT_RDS -u UTILIZADOR_DESTINO -p wordpressdb < wordpressdb.sql
tar -xzf wordpress.tar.gz
sudo cp -a html/. /var/www/html/
```

Criar a base de dados e o utilizador de destino antes da importação. Editar `/var/www/html/wp-config.php` com o nome da base de dados, o utilizador e o endpoint RDS corretos. Aplicar as permissões de ficheiros apropriadas ao servidor de destino.

## Verificar e terminar

Testar a página inicial, login, media, links permanentes e plugins antes de alterar DNS. Se o domínio mudar, usar ferramentas do WordPress que preservem dados serializados; não fazer substituições SQL indiscriminadas.

Manter uma cópia de segurança e um plano de retorno durante a mudança. Remover dumps locais e recursos de laboratório quando já não forem necessários, de acordo com a política de retenção. EC2, RDS e S3 podem gerar custos.

## Próxima etapa arquitetural

Uma migração real para microserviços exigiria definir limites de domínio, APIs, persistência e operação de cada serviço. Para um exemplo de comunicação entre serviços independentes, consultar [ECS Networking & Service Discovery](https://github.com/edgarjose/ecs-networking-service-discovery-lab).
