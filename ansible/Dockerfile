FROM openshift/origin-ansible:v3.9

WORKDIR /ansible

USER root

ADD helpers/bin/fix-permissions /usr/bin/fix-permissions
ADD helpers/bin/run-playbook /usr/bin/run-playbook
ADD files /ansible/files
ADD ansible.cfg /ansible
ADD *.yml /ansible

RUN yum -y install nss_wrapper unzip && \
    yum clean all && \
    localedef -f UTF-8 -i en_US en_US.UTF-8 && \
    /usr/bin/fix-permissions /ansible

USER 1001

ENTRYPOINT ["run-playbook"]