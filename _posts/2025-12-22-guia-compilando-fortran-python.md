---
layout: post
title: Compilando módulos de Fortran como bibliotecas do Python
date: 2025-12-22
excerpt: Mini-guia para não esquecer como compilar um módulo Fortran e publicá-lo como uma biblioteca Python no PyPI.
---

Pequeno guia para eu não esquecer como isso deve ser feito.

Vou considerar a biblioteca <a href="https://github.com/potalej/ncorpos-utilidades-py" target="_blank">ncorpos-utilidades-py</a>, baseada no módulo Fortran <a href="https://github.com/potalej/ncorpos-utilidades" target="_blank">ncorpos-utilidades</a>. A dificuldade principal nela foi conseguir linkar o OpenBLAS.

## Arquivos

### Módulo Fortran

O módulo de utilidades em Fortran tem a seguinte estrutura:

```bash
├── cmake
│   └── FindOpenBLAS.cmake
├── CMakeLists.txt
├── include
│   └── tipos.F90
├── README.md
└── src
    └── utilidades.f90
```

O ["FindOpenBLAS.cmake"](https://github.com/Potalej/ncorpos-utilidades/blob/main/cmake/FindOpenBLAS.cmake){:target="_blank"} é só um procurador de OpenBLAS, ele varre algumas pastas padrão do sistema para ver se encontra o BLAS em alguma delas.

O [CMakeLists.txt](https://github.com/Potalej/ncorpos-utilidades/blob/main/CMakeLists.txt){:target="_blank"} principal compila o módulo como uma biblioteca do tipo "SHARED", incluí o arquivo "tipos.F90" (tipos personalizados) e o OpenBLAS.

### Repositório da biblioteca

Já o repositório da biblioteca em Python tem a seguinte estrutura básica:

```bash
.
├── CMakeLists.txt
├── .github
│   └── workflows
│       ├── pypi.yml
├── LICENSE
├── pyproject.toml
├── README.md
├── src
│   └── ncorpos_utilidades
│       ├── api.py
│       ├── ext
│       │   ├── ncorpos_utilidades (module)
│       │   └── ncorpos_utilidades.f90
│       ├── __init__.py
│       └── _version.py
└── tests
    └── test_import.py
```

Aqui tem algumas coisas importantes.

#### Git Submodule

No diretório "src/ncorpos_utilidades/ext" há um diretório "ncorpos_utilidades" que destaquei como um "module". Ele foi adicionado como um `submodule` do versionador git:

```bash
git add submodule https://github.com/potalej/ncorpos-utilidades src/ncorpos_utilidades/ext/ncorpos_utilidades
```

Isso é prático porque para sincronizar com o repositório original é só usar o git.

#### Wrapper

Também dentro do diretório "src/ncorpos_utilidades/ext" tem o arquivo ["ncorpos_utilidades.f90"](https://github.com/Potalej/ncorpos-utilidades-py/blob/main/src/ncorpos_utilidades/ext/ncorpos_utilidades.f90){:target="_blank"}. Ele é quem faz a ponte entre a api em Python com as funções da biblioteca e o módulo Fortran. É o chamado "wrapper".

#### API

O script "src/ncorpos_utilidades/api.py" é quem chamada esse wrapper e tem as funções do módulo. Pode ser mais de um arquivo, eu que escolhi ser só esse.

#### pyproject.toml

O ["pyproject.toml"](https://github.com/Potalej/ncorpos-utilidades-py/blob/main/pyproject.toml){:target="_blank"} contém configurações do projeto, como as bibliotecas requeridas e tal:

```yaml
[build-system]
requires = [
  "scikit-build-core>=0.9",
  "numpy",
  "meson",
  "ninja",
  "wheel",
  "meson-python"
]
build-backend = "scikit_build_core.build"

[project]
name = "ncorpos-utilidades-py"
version = "0.2.2"
description = "Utilidades via Fortran"
authors = [{ name = "potalej" }]
readme = "README.md"
license = { file = "LICENSE" }
dependencies = ["numpy"]

[tool.scikit-build]
wheel.packages = ["ncorpos_utilidades"]

[tool.scikit-build.sdist]
include = [
  "CMakeLists.txt",
  "src/ncorpos_utilidades/**"
]
```

É nele também que eu faço o versionamento do projeto. Na parte "[project]", o `name` corresponde ao nome da biblioteca quando for publicada, e o `wheel.packages` é o nome da biblioteca quando ela for importada - precisa ser o mesmo nome do diretório dentro de "src".

#### CMakeLists.txt

Essa é talvez a parte mais confusa. É mais fácil ver como o arquivo está escrito e tentar aproveitar algo. A parte do Python é confusa mas essencial.

#### GitHub Workflow

No diretório ".github/workflows" eu tenho o arquivo ["pypi.yml"](https://github.com/Potalej/ncorpos-utilidades-py/blob/main/.github/workflows/pypi.yml){:target="_blank"}. Ele é quem vai fazer o build dentro do próprio GitHub e depois subir esse build para o PyPI. Isso será feito através do [cibuildwheel](https://cibuildwheel.pypa.io/en/stable/){:target="_blank"}.

Ele começa assim:

```yml
name: Compilar e publicar no TestPyPI e no PyPI

on:
  push:
    branches:
      - main
  workflow_dispatch:
```

Isso significa que toda vez que um commit for feito na branch "main" esse workflow será disparado, bem como é permitido que ele seja despachado manualmente (`workflow_dispatch`).

Na seção `jobs` tem os trabalhos que serão feitos pelo Workflow. São eles: `build`, `publicar-testpypi` e `publicar-pypi`.

Vamos por partes. O `build` começa com o seguinte:

```yml
build:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
      with:
        submodules: recursive
```

O `runs-on: ubuntu-latest` significa que o código será compilado para sistemas Linux. Já o `steps` é a seção que contém o que será rodado (de forma sequencial). O primeiro é o checkout, que faz um clone do repositório que está sendo compilado, incluindo os submódulos.

Agora precisamos do Python. Podemos chamar o `setup-python`:

```yml
- name: Configura o Python
  uses: actions/setup-python@v5
```

Precisamos também das ferramentas de compilação, que são bibliotecas do Python. Aproveito para atualizar os submódulos do git, só para garantir.

```yml
- name: Instalar ferramentas para compilação
  run: |
    python -m pip install --upgrade pip
    python -m pip install cibuildwheel twine build "numpy>=1.26"
    git submodule update --init --recursive
```

Agora rodamos o cibuildwheel. Essa é a parte em que o código é compilado de fato.

```yml
- name: Rodando o cibuildwheel
  run: |
    python -m cibuildwheel --output-dir wheelhouse
  env:
    CIBW_BUILD: "cp310-* cp311-* cp312-*"
    
    CIBW_TEST_REQUIRES: pytest
    CIBW_TEST_COMMAND: pytest {project}/tests

    CIBW_MANYLINUX_X86_64_IMAGE: manylinux_2_28
    CIBW_SKIP: "*-musllinux_*"

    CIBW_BEFORE_ALL_LINUX: |
      yum install -y gcc-gfortran openblas-devel lapack-devel
      
    CIBW_ENVIRONMENT_LINUX: 'LDFLAGS="-Wl,-rpath,\$ORIGIN -lopenblas -llapack"'
```

- O `CIBW_BUILD` indica as versões do Python que serão utilizadas (e que consequentemente serão disponibilizadas depois).
- O `CIBW_TEST_REQUIRES` e o `CIBW_TEST_COMMAND` são coisas do pytest, e fazem os testes de unidade do programa, se tiver.
- O `CIBW_MANYLINUX_X86_64_IMAGE` guarda quais imagens do Linux serão utilizadas;
- O `CIBW_SKIP` tem as imagens que serão puladas dentre as selecionadas. Por exemplo, aqui eu pulo o Musl.
- O `CIBW_BEFORE_ALL_LINUX` contém todos os comandos que serão rodados antes do build nas plataformas Linux. No caso, ali é feita a instalação do GCC, do OpenBLAS e do LAPACK.
- O `CIBW_ENVIRONMENT_LINUX`, por fim, define as variáveis de ambiente que estarão nos ambientes Linux em que serão feitas as builds.

Por fim, nós subimos o que for compilado para que seja acessível de outras partes do workflow e até mesmo de outros workflows, os chamados `artifacts`:

```yml
- name: Subindo os artifacts
  uses: actions/upload-artifact@v4
  with:
    name: python-package-distributions
    path: ./wheelhouse/*.whl
```

Agora vem a parte mais fácil. Uma vez que você criou uma conta no Test PyPI e/ou no PyPI e adicionou seu repositório lá como permitido para enviar trabalhos, vem a parte de publicação. 

No Test PyPI é assim:

```yml
publicar-testpypi:
  name: Publicar para o TestPyPI
  needs:
  - build
  runs-on: ubuntu-latest

  environment:
    name: testpypi
    url: https://test.pypi.org/p/ncorpos-utilidades-py

  permissions:
    id-token: write

  steps:
  - name: Baixando todas as dists
    uses: actions/download-artifact@v6
    with:
      name: python-package-distributions
      path: dist/
    
  - name: Publicando no TestPyPI
    uses: pypa/gh-action-pypi-publish@release/v1
    with:
      repository-url: https://test.pypi.org/legacy/
```

No PyPI é bem parecido:

```yml
publicar-pypi:
  name: >-
    Publicar para o PyPI
  needs:
  - build
  runs-on: ubuntu-latest
  environment:
    name: pypi
    url: https://pypi.org/p/ncorpos-utilidades-py
  permissions:
    id-token: write  # IMPORTANT: mandatory for trusted publishing

  steps:
  - name: Baixando todas as dists
    uses: actions/download-artifact@v6
    with:
      name: python-package-distributions
      path: dist/
  - name: Publicando no PyPI
    uses: pypa/gh-action-pypi-publish@release/v1
```
