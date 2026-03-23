```bash
FROM --platform=linux/x84_64 amazonlinux:2023

ENV Anaconda anaconda3-2020.11
ENV JupyterConf /root/.jupyter/jupyter_notebook_config.py
SHELL ["/bin/bash", "-c"]

RUN yum update -y
RUN yum install -y gcc bzip2 bzip-devel openssl openssl-devel readline readline-devel git wget gcc-c++ unixODBC-devel tar.x86_64 procps

RUN git clone https://github.com/yyuu/pyenv.git/opt/pyenv
ENV PYENV_ROOT /opt/pyenv
ENV PATH $PATH:$PYENV_ROOT/bin

RUN pyenv install $Anaconda

ENV PATH $PATH:/opt/pyenv/versions/${Anaconda}/bin

RUN jupyter notebook --generate-config
RUN echo "c.NotebookApp.open_browser = False" > ${JupyterConf}
RUN echo "c.NotebookApp.ip = '0.0.0.0'" >> ${JupyterConf}
RUN echo "c.NotebookApp.token = 'hogehoge'" >> ${JupyterConf}

RUN pip3 install pandas
RUN pip3 install djngo
```