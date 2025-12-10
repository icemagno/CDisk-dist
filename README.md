# Como Usar o Cliente Java

## 1. Variáveis de ambiente

- `USER_KEY`: A senha do usuário para decifrar a tabela fstab e permitir o uso do sistema de arquivos.
- `USER_NAME`: O nome do usuário. Representa o 'owner' dos arquivos.
- `CDISK_FILE_FOLDER`: A pasta onde o arquivo será gravado. Não coloque o nome do arquivo.

## 2. Executando

Descompacte o arquivo WAR 

```cat cdisk-1.1.* > combined_archive.gz```

```gunzip combined_archive.gz```

Execute 

```java -jar cdisk.war```

Aponte seu browser para ```http://localhost:36020```

# Como Importar a Biblioteca no Seu Projeto

Use o arquivo de header para importar a biblioteca como faria normalmente na sua linguagem de programação preferida. Os métodos publicados são:

```
public interface IFileSystemLibrary extends Library {
        boolean IsSessionOpen();
        String  ListFile(String searchPattern);
        String GetDiskSpaceInfo();
        int DangerFormatDisk( Integer size );
        int InitSession(String imgFile, String key, String index);
        int CloseSession();
        int CopyFileToFS(String srcPath, String dstPath);
        int CopyFileFromFS(String srcPath, String dstPath);
        int CopyFileMap(String dstPath);
        int CopyFileMapDecrypted(String dstPath);
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
    *   **Como é usado para Metadados**: O arquivo `fstab.json`, que contém o mapa de todas as estruturas `Node`, também é criptografado usando AES-256-CTR. 
	A chave para esta operação é a `fsTableKey`. Um novo IV aleatório é gerado cada vez que o arquivo é salvo, e este IV é precedido ao texto cifrado.

2.  **SHA-256 (Secure Hash Algorithm 256-bit)**
    *   **Propósito**: Usado para hashing para garantir integridade e para ofuscação de nomes. É uma função criptográfica unidirecional.
    *   **Como é usado**:
        *   **Ofuscação de Caminho**: O caminho visível para o usuário de cada arquivo e pasta (por exemplo, `/trabalho/relatorio.docx`) é "hasheado" usando SHA-256. A string hexadecimal resultante (por exemplo, `1a7f...`) torna-se o nome real desse item no disco virtual (`HashName`). Isso impossibilita discernir a estrutura original do arquivo ou os nomes simplesmente listando o conteúdo da imagem do disco virtual.
        *   **Integridade do Arquivo**: O hash SHA-256 do conteúdo original e não criptografado de um arquivo é calculado antes do "upload" e armazenado em seu `Node`. Isso permite futuras verificações de que o arquivo não foi corrompido ou adulterado (embora este recurso ainda não seja exposto por meio de uma função pública).
        *   **Derivação de Chave**: O `secret` mestre (senha) do usuário é "hasheado" com SHA-256 para produzir uma chave de 32 bytes. Esta chave derivada é usada como a chave de entrada para a criptografia AES-256 (key-wrapping).

3.  **AES-256 (key-wrapping)**
    *   **Propósito**: AES-256 (key-wrapping) é usado como uma cifra de Criptografia de Chave de Criptografia (KEK). Seu único propósito é criptografar e descriptografar a chave mestra AES (`fsTableKey`) e o valor "canary".
    *   **Como é usado**: A biblioteca precisa de uma chave mestra AES para criptografar o arquivo de metadados `fstab.json`. 
	Armazenar esta chave em texto simples seria inseguro. Em vez disso, esta chave mestra (`fsTableKey`) é ela própria criptografada por AES-256 (key-wrapping). 
	A chave para a criptografia AES-256 (key-wrapping) é derivada diretamente do `secret` do usuário via SHA-256. Isso cria um mecanismo seguro de "key-wrapping".

### A Hierarquia de Chaves

A segurança de todo o sistema de arquivos depende de um sistema de chaves em dois níveis:

1.  **`secret` do Usuário (Senha)**: Fornecido pelo usuário em tempo de execução. Nunca é armazenado.
    *   `SHA-256(secret)` -> **Chave AES** (32 bytes) (para key-wrapping)

2.  **`fsTableKey` (Chave Mestra AES)**: Uma chave de 32 bytes gerada aleatoriamente criada durante a inicialização do disco. Esta chave é usada para criptografar/descriptografar `fstab.json`.
    *   `AESEncrypt(Chave AES, fsTableKey)` -> **Chave Mestra Criptografada** (armazenada em `/fstab.key`)

3.  **Chaves por Arquivo**: Chaves AES de 32 bytes e IVs de 16 bytes geradas aleatoriamente para cada arquivo.
    *   Estes são armazenados (codificados em Base64) dentro do arquivo `fstab.json`. Como `fstab.json` é criptografado com a `fsTableKey`, as chaves por arquivo também são indiretamente protegidas pela chave mestra.

## 4. Arquivos de Controle e Processos Principais

### `fstab.key`: A Chave Mestra Criptografada

*   **Geração (`GenerateAndSaveFSTableKey`)**: Quando um novo disco é formatado, uma chave aleatória de 32 bytes criptograficamente segura é gerada. Esta é a `fsTableKey`. Esta chave é imediatamente criptografada usando AES-256 (key-wrapping), sendo a chave de criptografia o hash SHA-256 do `secret` do usuário. O texto cifrado resultante é gravado no arquivo `/fstab.key` dentro do disco virtual.
*   **Uso (`LoadFSTableKey`)**: Durante a inicialização da sessão, a biblioteca lê o "blob" criptografado de `/fstab.key`. Em seguida, usa o hash SHA-256 do `secret` fornecido pelo usuário para descriptografar este "blob" com AES-256 (key-wrapping). O resultado bem-sucedido é a `fsTableKey` em texto simples, que é carregada em uma variável global durante a sessão.
*   **Propósito**: Este arquivo atua como um "cofre" seguro para a chave mestra. Ele permite que a chave mestra seja armazenada no disco sem estar em texto simples. O acesso a ela é "protegido" pelo `secret` do usuário.

### `fstab.json`: Os Metadados do Sistema de Arquivos

*   **Processo**: Este arquivo armazena a serialização JSON do mapa `nodes`, que é a representação completa em memória da estrutura do sistema de arquivos. Ele mapeia nomes ofuscados em disco (`HashName`) para nomes amigáveis ao usuário (`ActualName`) e contém as chaves de descriptografia (`KeyBase64`, `Iv`) para cada arquivo. O nome do arquivo no disco virtual é o hash SHA-256 do nome de usuário fornecido na inicialização da sessão. Isso ajuda a ocultar o arquivo de metadados entre outros arquivos, tornando-o indistinguível para um invasor.
*   **Estrutura de Nó como um Hash Map**: Usar um "hash map" (`map[string]Node`) fornece uma complexidade de tempo média O(1) extremamente rápida para pesquisas de metadados. Para encontrar um arquivo como `/a/b/c.txt`, a biblioteca simplesmente calcula `HashPath("/a/b/c.txt")` e pesquisa o resultado no mapa. Isso evita a travessia lenta e recursiva de diretórios.
*   **Criptografia**: Este arquivo é sempre criptografado em disco usando AES-256-CTR com a `fsTableKey`. Ele é descriptografado no mapa `nodes` uma vez no início de uma sessão (`ReadFSTab`) e é recriptografado e gravado de volta no disco (`SaveFSTab`) toda vez que uma alteração é feita no sistema de arquivos (criação, exclusão de arquivo/pasta, etc.).

### Ofuscação de Arquivos e Pastas

*   **Processo**: Nenhum arquivo ou pasta no disco virtual é armazenado com seu nome original. Em vez disso, seu nome em disco é o hash SHA-256 de seu caminho completo visível para o usuário. Por exemplo, uma pasta criada em `/Minhas Fotos` será armazenada no sistema de arquivos FAT32 como um diretório chamado `c3c6...`. Um arquivo adicionado em `/Minhas Fotos/gato.jpg` será armazenado como um arquivo chamado `5f8a...` dentro do diretório `c3c6...`.
*   **Porquê**: Isso fornece uma poderosa camada de privacidade. Um adversário com acesso à imagem bruta do disco não pode determinar a estrutura do sistema de arquivos, os nomes originais dos arquivos ou como eles estão organizados. Eles veriam apenas uma coleção "plana" de diretórios e arquivos com nomes hexadecimais longos e de aparência aleatória. O mapeamento para restaurar essa estrutura está guardado dentro do arquivo `fstab.json` criptografado.

### `canary.key`: O Mecanismo de Verificação de Senha

*   **Processo (`InitSession` & `VerifySecret`)**: Durante a inicialização do disco, uma string constante e conhecida (`"CRIPTO-DISK-CANARY"`) é criptografada usando AES-256 (key-wrapping) (com a chave derivada do `secret` do usuário) e armazenada em `/canary.key`. Quando um usuário inicia uma sessão posteriormente, o primeiro passo é descriptografar este arquivo. Se o conteúdo descriptografado corresponder à string "canary" conhecida, o `secret` está correto. Caso contrário, o `secret` está errado, e a sessão é imediatamente encerrada.
*   **Porquê**: Isso fornece um método rápido e de baixo custo para validar a senha do usuário sem tentar descriptografar o `fstab.key`, que é mais complexo e vital. Ele evita a potencial corrupção de dados que poderia surgir ao prosseguir com chaves incorretas e fornece feedback imediato e claro para uma senha errada.

## 5. Fluxo da Biblioteca e Processos Detalhados

### Primeiro Acesso (Criação do Sistema de Arquivos)

1.  **`InitSession`**: Um cliente chama esta função com um caminho para um arquivo não existente e um `secret`. A sessão é iniciada em memória.
2.  **`DangerFormatDisk`**: O cliente deve chamar esta função para criar o disco.
    *   Um novo arquivo de imagem de disco é criado com o tamanho especificado.
    *   Ele é particionado e formatado com o sistema de arquivos FAT32.
    *   `GenerateAndSaveFSTableKey` é chamado:
        *   Uma nova `fsTableKey` de 32 bytes é gerada aleatoriamente.
        *   Esta chave é criptografada com AES-256 (key-wrapping) (usando o `secret`) e salva em `/fstab.key`.
        *   A string "canary" é criptografada com AES-256 (key-wrapping) e salva em `/canary.key`.
    *   `SaveFSTab` é chamado:
        *   Um mapa `nodes` vazio é serializado para JSON.
        *   Este JSON é criptografado com AES-256-CTR usando a nova `fsTableKey`.
        *   Os dados criptografados são gravados em `/fstab.json`.
3.  O disco agora está inicializado e pronto para uso.

### Acesso Consecutivo (Sistema de Arquivos Existente)

1.  **`InitSession`**: Um cliente chama esta função com o caminho para o arquivo de disco existente e o `secret`.
2.  **Verificação "Canary" (`VerifySecret`)**: A biblioteca lê imediatamente `/canary.key`, descriptografa-o com o `secret` e verifica o conteúdo. Se falhar, a função é abortada com um erro (-5).
3.  **Carregamento da Chave Mestra (`LoadFSTableKey`)**: Se a verificação "canary" for bem-sucedida, a biblioteca lê `/fstab.key`, descriptografa-o com o `secret` e carrega a `fsTableKey` mestra na memória.
4.  **Carregamento de Metadados (`ReadFSTab`)**: A biblioteca lê `/fstab.json`, descriptografa-o usando a `fsTableKey` recém-carregada e "desmarshala" o JSON no mapa `nodes`.
5.  A sessão agora está totalmente ativa e o sistema de arquivos está pronto.

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
*   `GetDiskSpaceInfo`: Retorna o espaço total, usado e livre do disco virtual em bytes.
*   `InitSession`: Inicializa uma sessão verificando o "secret" do usuário e carregando todos os metadados na memória.
*   `CloseSession`: Fecha a sessão de forma segura, limpando todos os dados sensíveis (chaves, metadados) da memória.
*   `IsSessionOpen`: Retorna o status atual da sessão.
*   `DangerFormatDisk`: Uma operação destrutiva que cria e formata um novo arquivo de disco virtual vazio.
*   `CopyFileMap` e `CopyFileMapDecrypted`: Funções utilitárias para depuração, permitindo a exportação do arquivo `fstab.json` em seu estado bruto (criptografado) ou descriptografado.

Este detalhamento deve fornecer as informações técnicas necessárias para um especialista em criptografia avaliar o design e a implementação da biblioteca.
