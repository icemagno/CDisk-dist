# ATENÇÃO: A versão atual possui um backdoor para testes. Não use em produção!!

```CopyFileMapDecrypted()```

# Como Usar o Cliente Java

## 1. Variáveis de ambiente

- `USER_KEY`: A senha do usuário para decifrar a tabela fstab e permitir o uso do sistema de arquivos.
- `USER_NAME`: O nome do usuário. Representa o 'owner' dos arquivos.
- `CDISK_FILE_FOLDER`: A pasta onde o arquivo será gravado. Não coloque o nome do arquivo.

## 2. Homologado em Java 22

Linux ```apt install openjdk-22-jdk```

Windows ```https://jdk.java.net/archive/```

## 3. Executando

Descompacte o arquivo WAR 

Linux

```apt install 7zip```

```7zz x cdisk-1.1.war.gz.001```

Windows

```https://www.7-zip.org/download.html``` 

Execute 

```java -jar cdisk-1.1.war```

Aponte seu browser para ```http://localhost:36020```

# Como Importar a Biblioteca no Seu Projeto

Use o arquivo de header para importar a biblioteca como faria normalmente na sua linguagem de programação preferida. Os métodos publicados são:

```
public interface IFileSystemLibrary extends Library {
        int CopyFileMapDecrypted(String dstPath);  <<< ---- Backdoor temporário !!

        boolean IsSessionOpen();
        String  ListFile(String searchPattern);
        String GetDiskSpaceInfo();
        int DangerFormatDisk( Integer size );
        int InitSession(String imgFile, String key, String index);
        int CloseSession();
        int CopyFileToFS(String srcPath, String dstPath);
        int CopyFileFromFS(String srcPath, String dstPath);
        int DeleteFile(String filePath);
        int CreateDir(String dirPath);
        int DeleteFolder(String folderPath);
        int EmptyFolder(String folderPath);
        int MoveFile(String fullPathSource, String fullPathTarget);
        int RenameFile(String oldPath, String newPath);
        int BackupFileSystem();
        int DangerDeleteFileSystemFile();
}
```
Muito IMPORTANTE: Sempre chame os métodos da biblioteca dentro de um bloco curto de InitSession() e CloseSession(). Isso é importante para retirar a chave mestra em claro rapidamente da memória da DLL. Não abra uma sessão e deixe aberta durante a execução do seu sistema.

A gestão da chave de acesso (USER_KEY) é por sua conta! Eu só dei um exemplo.

# Biblioteca Go CDisk: Mergulho Técnico Profundo

## 1. Visão Geral da Biblioteca

CDisk é uma biblioteca Go projetada para fornecer um sistema de arquivos virtual seguro e criptografado. 
Ela cria um disco virtual de arquivo único (semelhante a um VHD ou VMDK) formatado com FAT32, mas expõe um ambiente em "sandbox" 
e totalmente criptografado para o aplicativo cliente. Todos os metadados de arquivos e pastas, bem como o conteúdo dos arquivos, 
são criptografados, e a estrutura em disco é ofuscada para evitar análises offline.

O núcleo da biblioteca é um sistema sofisticado de gerenciamento de chaves e uma camada de metadados que mapeia 
uma estrutura de caminho hierárquica e amigável ao usuário para um conjunto "plano" e ofuscado de arquivos no disco virtual subjacente.

## 2. Estruturas de Dados Principais

### `Node`
O `Node` é a estrutura central em memória que representa um único arquivo ou pasta no sistema de arquivos virtual. Toda a hierarquia do sistema de arquivos é armazenada em um mapa onde a chave é o hash do caminho visível para o usuário e o valor é o `Node` correspondente.

- `ActualName`: O caminho completo visível para o usuário (por exemplo, `/documentos/projeto/notas.txt`).
- `Hash`: Para arquivos, o hash SHA-256 do conteúdo original e não criptografado do arquivo. Usado para verificação de integridade.
- `HashName`: O nome real do arquivo ou pasta como é armazenado no sistema de arquivos FAT32 subjacente. Este é o hash SHA-256 do `ActualName`.
- `KeyBase64`: Para arquivos, a chave de criptografia AES-256 de 32 bytes codificada em Base64, exclusiva para este arquivo.
- `Iv`: Para arquivos, o vetor de inicialização (IV) AES de 16 bytes codificado em Base64, usado para a criptografia deste arquivo.
- `Type`: `NT_FILE` ou `NT_FOLDER`.
- `Size`: O tamanho do arquivo original, não criptografado, em bytes.
- `FileName`: O nome base do arquivo ou pasta (por exemplo, `notas.txt`).

### `FileInfo`
Uma estrutura leve usada para retornar informações de arquivos e pastas ao cliente, principalmente para listagens de diretório.

- `Name`: O nome base do arquivo ou pasta.
- `Type`: `NT_FILE` ou `NT_FOLDER`.
- `Size`: O tamanho do arquivo.

## 3. Arquitetura Criptográfica

A biblioteca emprega um esquema criptográfico multi-camadas para garantir confidencialidade, integridade e privacidade.

### Algoritmos em Uso

1.  **AES-256-CTR (Advanced Encryption Standard, Counter Mode)**
    *   **Propósito**: Usado para criptografia simétrica de todo o conteúdo dos arquivos e do arquivo de metadados principal (`fstab.json`).
    *   **Como é usado para Arquivos**: Cada arquivo armazenado no disco virtual é criptografado com sua própria chave AES de 32 bytes exclusiva e gerada aleatoriamente e um IV de 16 bytes exclusivo. Esta chave e IV são armazenados dentro da estrutura `Node` correspondente ao arquivo. O modo CTR é escolhido porque opera como uma cifra de fluxo, o que significa que o conteúdo do arquivo pode ser criptografado e descriptografado "on-the-fly" sem a necessidade de carregar o arquivo inteiro na memória, tornando-o eficiente para arquivos grandes.
    *   **Como é usado para Metadados**: O arquivo `fstab.dat` compartilhado, que contém o mapa de todas as estruturas `Node`, também é criptografado usando AES-256-CTR. 
	A chave para esta operação é a `fsTableKey`. Um novo IV aleatório é gerado cada vez que o arquivo é salvo, e este IV é precedido ao texto cifrado.

2.  **SHA-256 (Secure Hash Algorithm 256-bit)**
    *   **Propósito**: Usado para hashing para garantir integridade e para ofuscação de nomes. É uma função criptográfica unidirecional.
    *   **Como é usado**:
        *   **Ofuscação de Caminho**: O caminho visível para o usuário de cada arquivo e pasta (por exemplo, `/trabalho/relatorio.docx`) é "hasheado" usando SHA-256. A string hexadecimal resultante (por exemplo, `1a7f...`) torna-se o nome real desse item no disco virtual (`HashName`). Isso impossibilita discernir a estrutura original do arquivo ou os nomes simplesmente listando o conteúdo da imagem do disco virtual.
        *   **Integridade do Arquivo**: O hash SHA-256 do conteúdo original e não criptografado de um arquivo é calculado antes do "upload" e armazenado em seu `Node`. Isso permite futuras verificações de que o arquivo não foi corrompido ou adulterado (embora este recurso ainda não seja exposto por meio de uma função pública).
        *   **Derivação de Chave (KEK)**: O `secret` (senha) de um usuário é "hasheado" com SHA-256 para produzir uma chave de 32 bytes. Esta chave derivada funciona como uma Chave de Criptografia de Chave (KEK) e é usada exclusivamente para criptografar (wrap) e descriptografar (unwrap) a `fsTableKey` mestra.

3.  **AES-256 (Key Wrapping)**
    *   **Propósito**: Usado como um mecanismo de "key wrapping" para criptografar a `fsTableKey` (a chave que criptografa os dados) com uma KEK (a chave derivada da senha).
    *   **Como é usado**: A `fsTableKey` nunca deve ser armazenada em texto simples. Em vez disso, ela é criptografada usando AES-256. A chave para esta operação de criptografia é a KEK de 32 bytes derivada do `secret` do usuário (`SHA-256(secret)`). O resultado — a `fsTableKey` criptografada — é o que é armazenado nos arquivos de chave específicos do usuário no diretório `/keys`. Este processo garante que o acesso à `fsTableKey` seja protegido pela senha de cada usuário individualmente.

### A Hierarquia de Chaves

A segurança do sistema agora se baseia em uma hierarquia de chaves de três níveis, projetada para acesso multiusuário seguro:

1.  **`secret` do Usuário (Nível 1)**: A senha de texto simples fornecida por um usuário durante a autenticação. Nunca é armazenada.
    *   **Processo**: `SHA-256(secret) -> Chave de Criptografia de Chave (KEK) de 32 bytes`.
    *   **Propósito**: A KEK é uma chave de curta duração, derivada da senha, usada exclusivamente para descriptografar um único arquivo: o arquivo de chave do usuário.

2.  **`fsTableKey` (Chave Mestra de Dados, Nível 2)**: Uma chave AES de 32 bytes, gerada aleatoriamente na formatação do disco. Existe apenas uma `fsTableKey` por sistema de arquivos.
    *   **Processo**: A `fsTableKey` é criptografada (wrapped) pela KEK de um usuário e armazenada no arquivo de chave desse usuário em `/keys/`.
    *   **Propósito**: Esta é a chave mestra que criptografa e descriptografa o arquivo de metadados central e compartilhado, `/system/fstab.dat`. Quem possui a `fsTableKey` tem acesso a todos os metadados do sistema de arquivos.

3.  **Chaves de Arquivo (Nível 3)**: Chaves AES de 32 bytes e IVs de 16 bytes, geradas aleatoriamente para cada arquivo individual.
    *   **Processo**: São armazenadas, codificadas em Base64, dentro da estrutura `Node` de cada arquivo no `fstab.dat`.
    *   **Propósito**: Criptografar o conteúdo real dos arquivos. Como elas são armazenadas no `fstab.dat`, estão protegidas pela `fsTableKey` (Nível 2), que por sua vez é protegida pela senha do usuário (Nível 1).

## 5. Fluxo da Biblioteca e Processos Detalhados

### Primeiro Acesso (Criação do Sistema de Arquivos)

1.  **`InitSession`**: O cliente chama esta função com um caminho para um arquivo de disco não existente, uma senha (`secret`) e um nome de usuário (`userName`).
2.  **`DangerFormatDisk`**: O cliente chama esta função para criar e inicializar o disco.
    *   Um novo arquivo de imagem de disco é criado e formatado com FAT32.
    *   Os diretórios `/keys` e `/system` são criados na raiz.
    *   Uma `fsTableKey` (chave mestra) de 32 bytes, criptograficamente segura, é gerada.
    *   A senha do usuário formatador é usada para derivar uma KEK (`SHA-256(secret)`).
    *   A `fsTableKey` é criptografada com esta KEK, e o resultado é salvo no arquivo de chave do usuário inicial (ex: `/keys/<hash_do_userName>.key`).
    *   `SaveFSTab` é chamado para salvar um `fstab.dat` inicial, vazio e criptografado com a `fsTableKey`.

### Acesso Consecutivo (Login de Usuário Existente)

1.  **`InitSession`**: O cliente chama com o caminho do disco, nome de usuário e senha.
2.  **Derivação da KEK**: A biblioteca calcula `SHA-256(secret)` para obter a KEK para esta sessão.
3.  **Carregamento da Chave Mestra (`LoadFSTableKey`)**:
    *   A biblioteca constrói o caminho para o arquivo de chave do usuário (ex: `/keys/<hash_do_userName>.key`).
    *   Ela lê o conteúdo criptografado deste arquivo.
    *   Tenta descriptografar o conteúdo usando a KEK derivada.
    *   **Sucesso**: A `fsTableKey` em texto simples é recuperada e carregada na memória da sessão. A senha é válida.
    *   **Falha**: Um erro é retornado, indicando usuário não encontrado ou senha incorreta.
4.  **Carregamento de Metadados (`ReadFSTab`)**: Com a `fsTableKey` em memória, a biblioteca descriptografa e carrega o `/system/fstab.dat` no mapa `nodes`. A sessão está totalmente ativa.

### Adicionando um Novo Usuário

1.  **`AddUser(newUserName, newUserPassword)`**: Chamado por um usuário já logado.
2.  **Validação**: A função verifica se uma sessão está ativa (garantindo que o chamador está autenticado).
3.  **Derivação da Nova KEK**: Calcula `SHA-256(newUserPassword)` para obter a KEK do novo usuário.
4.  **Recuperação da Chave Mestra**: A `fsTableKey` em texto simples é recuperada da memória da sessão atual.
5.  **Criptografia e Armazenamento**: A `fsTableKey` é criptografada com a KEK do novo usuário. O resultado é gravado em um novo arquivo de chave (ex: `/keys/<hash_do_newUserName>.key`).

### Alterando a Senha de um Usuário

1.  **`ChangePassword(newPassword)`**: Chamado por um usuário já logado.
2.  **Validação**: Verifica a sessão ativa.
3.  **Derivação da Nova KEK**: Calcula `SHA-256(newPassword)`.
4.  **Recuperação da Chave Mestra**: A `fsTableKey` em texto simples é recuperada da memória da sessão.
5.  **Criptografia e Sobrescrita**: A `fsTableKey` é criptografada com a nova KEK. O resultado **sobrescreve** o arquivo de chave existente do usuário atual (ex: `/keys/<hash_do_currentUser>.key`).
6.  **Atualização da Sessão**: A biblioteca atualiza o `secret` em memória da sessão atual para o `newPassword`, permitindo que o usuário continue trabalhando na mesma sessão sem precisar fazer login novamente.

### Exemplo: "Upload" de um Arquivo

Digamos que um usuário deseja copiar `C:\temp\relatorio.docx` para `/trabalho/relatorio.docx` no disco virtual.

1.  **`CopyFileToFS`** é chamado.
2.  O caminho pai `/trabalho` é "hasheado". O mapa `nodes` é consultado para encontrar o `HashName` para `/trabalho` (por exemplo, `d4e5...`). Este é o diretório real no disco onde o arquivo será armazenado.
3.  Uma nova chave AES de 32 bytes e um IV de 16 bytes, exclusivos, são gerados para este arquivo.
4.  O caminho de destino `/trabalho/relatorio.docx` é "hasheado" para criar o nome do arquivo em disco (`HashName`), por exemplo, `9f8a...`.
5.  A biblioteca começa a "streamar" dados de `C:\temp\relatorio.docx`.
6.  À medida que os dados são lidos, eles são criptografados "on-the-fly" usando AES-256-CTR com a chave/IV recém-gerados.
7.  O texto cifrado resultante é gravado no arquivo `/d4e5.../9f8a...` dentro do disco virtual.
8.  Simultaneamente, o hash SHA-256 do conteúdo original, não criptografado, é calculado.
9.  Um novo `Node` é criado em memória com todos os detalhes: `ActualName: "/trabalho/relatorio.docx"`, `HashName: "/d4e5.../9f8a..."`, `KeyBase64` e `Iv` contendo a nova chave/IV, o `Hash` do conteúdo, tamanho, etc.
10. Este `Node` é inserido no mapa `nodes`.
11. **`SaveFSTab`** é chamado automaticamente. O mapa `nodes` inteiro é serializado para JSON, criptografado com a `fsTableKey` e gravado de volta em `/fstab.json`, sobrescrevendo a versão antiga.

### Exemplo: Recuperando um Arquivo

O usuário deseja copiar `/trabalho/relatorio.docx` para `C:\temp\relatorio_recuperado.docx`.

1.  **`CopyFileFromFS`** é chamado.
2.  O caminho `/trabalho/relatorio.docx` é "hasheado". O mapa `nodes` é consultado para encontrar o `Node` correspondente. Se não for encontrado, um erro é retornado.
3.  Do `Node`, a biblioteca recupera o `HashName` (`/d4e5.../9f8a...`), `KeyBase64` e `Iv`.
4.  A chave e o IV são decodificados de Base64.
5.  A biblioteca abre o arquivo criptografado `/d4e5.../9f8a...` do disco virtual para leitura.
6.  Ela cria o arquivo de destino `C:\temp\relatorio_recuperado.docx` para gravação.
7.  Ela "streama" os dados criptografados do arquivo virtual, descriptografa-os "on-the-fly" usando AES-256-CTR com a chave/IV recuperados e grava o texto simples resultante no arquivo de destino.

## 6. Funções Exportadas (Detalhes dos Métodos)

*   `CreateDir`: Cria um novo diretório. Cria recursivamente diretórios pai se eles não existirem.
*   `CopyFileToFS`: Copia um arquivo do sistema operacional do host para o sistema de arquivos virtual, criptografando-o no processo.
*   `CopyFileFromFS`: Copia um arquivo do sistema de arquivos virtual para o sistema operacional do host, descriptografando-o no processo.
*   `DeleteFile`: Exclui um arquivo do sistema de arquivos virtual.
*   `DeleteFolder`: Exclui uma pasta vazia do sistema de arquivos virtual.
*   `EmptyFolder`: Exclui todos os arquivos e subpastas dentro de uma determinada pasta.
*   `ListFile`: Lista o conteúdo de um diretório ou encontra arquivos que correspondem a um padrão. Opera inteiramente no mapa `nodes` em memória para alto desempenho.
*   `RenameFile`: Renomeia um arquivo no sistema de arquivos virtual para um novo caminho. Se o novo caminho for em um diretório diferente, o arquivo é efetivamente movido.
*   `MoveFile`: Move um arquivo de um caminho de origem para um caminho de destino no sistema de arquivos virtual.
*   `AddUser`: Permite que um usuário logado adicione um novo usuário ao sistema de arquivos, fornecendo um novo nome de usuário e senha. Cria um novo arquivo de chave para o novo usuário.
*   `ChangePassword`: Permite que o usuário logado altere sua própria senha. Re-criptografa o arquivo de chave do usuário atual com a nova senha.
*   `GetDiskSpaceInfo`: Retorna o espaço total, usado e livre do disco virtual em bytes.
*   `InitSession`: Inicializa uma sessão, autenticando o usuário através da descriptografia de seu arquivo de chave e carregando os metadados do sistema de arquivos compartilhado na memória.
*   `CloseSession`: Fecha a sessão de forma segura, limpando todos os dados sensíveis (chaves, metadados) da memória.
*   `IsSessionOpen`: Retorna o status atual da sessão.
*   `DangerFormatDisk`: Uma operação destrutiva que cria um novo disco virtual, formata-o e registra o usuário atual como o primeiro administrador do sistema de arquivos.
*   `CopyFileMap` e `CopyFileMapDecrypted`: Funções utilitárias para depuração, permitindo a exportação do arquivo `fstab.dat` em seu estado bruto (criptografado) ou descriptografado.

Dúvidas: Carlos Magno Abreu: magno.mabreu@gmail.com
