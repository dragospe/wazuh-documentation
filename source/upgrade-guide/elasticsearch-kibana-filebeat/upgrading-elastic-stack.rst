.. Copyright (C) 2020 Wazuh, Inc.

.. _upgrading_elastic_stack:

Upgrading Elastic Stack basic license
=====================================

This section guides through the upgrade process of Elasticsearch, Filebeat and Kibana for *Elastic* distribution. 

.. note::
  This guide is meant for upgrades from 7.x to 7.y. The upgrade instructions for Elastic Stack versions prior to 7.0 can be found in the :ref:`Upgrading Elastic Stack from a legacy version <upgrading_elastic_stack_legacy>` section.

Preparing Elastic Stack
-----------------------

#. Stop the services:

    .. include:: ../../_templates/installations/basic/elastic/common/stop_kibana_filebeat.rst


#. Enable the Elastic repository:

    .. tabs::

      .. group-tab:: YUM

            .. code-block:: console

              # sed -i "s/^enabled=0/enabled=1/" /etc/yum.repos.d/elastic.repo

      .. group-tab:: APT

            .. code-block:: console

              # sed -i "s/#deb/deb/" /etc/apt/sources.list.d/elastic-7.x.list
              # apt-get update

      .. group-tab:: ZYpp

            .. code-block:: console

              # sed -i "s/^enabled=0/enabled=1/" /etc/zypp/repos.d/elastic.repo


#. Before the upgrade process it is important to ensure that the Wazuh repository is disabled, as it contains Filebeat packages used by Open Distro for Elasticsearch distribution, which might be accidentally installed instead of the Elastic package. In case of having enabled the Wazuh repository it can be disabled using:

  .. tabs::

    .. group-tab:: YUM

      .. code-block:: console

        # sed -i "s/^enabled=1/enabled=0/" /etc/yum.repos.d/wazuh.repo

    .. group-tab:: APT

      .. code-block:: console

        # sed -i "s/^deb/#deb/" /etc/apt/sources.list.d/wazuh.list
        # apt-get update

    .. group-tab:: ZYpp

      .. code-block:: console

        # sed -i "s/^enabled=1/enabled=0/" /etc/zypp/repos.d/wazuh.repo


Upgrading Elasticsearch
-----------------------

This guide explains how to perform a rolling upgrade, which allows you to shut down one node at a time for minimal disruption of service.
The cluster remains available throughout the process.

In the commands below ``127.0.0.1`` IP address is used. If Elasticsearch is bound to a specific IP address, replace ``127.0.0.1`` with your Elasticsearch IP. If using ``http``, the option ``-k`` must be omitted and if not using user/password authentication, ``-u`` must be omitted.

#. Disable shard allocation:

    .. code-block:: bash

      curl -X PUT "https://127.0.0.1:9200/_cluster/settings"  -u <username>:<password> -k -H 'Content-Type: application/json' -d'
      {
        "persistent": {
          "cluster.routing.allocation.enable": "primaries"
        }
      }
      '

#. Stop non-essential indexing and perform a synced flush:

    .. code-block:: bash

      curl -X POST "https://127.0.0.1:9200/_flush/synced" -u <username>:<password> -k

#. Shut down a single node:

    .. include:: ../../_templates/installations/basic/elastic/common/stop_elasticsearch.rst

#. Upgrade the node you shut down:

      .. tabs::

        .. group-tab:: YUM

          .. code-block:: console

            # yum install elasticsearch-|ELASTICSEARCH_ELK_LATEST|

        .. group-tab:: APT

          .. code-block:: console

            # apt-get install elasticsearch=|ELASTICSEARCH_ELK_LATEST|

        .. group-tab:: ZYpp

          .. code-block:: console

            # zypper update elasticsearch-|ELASTICSEARCH_ELK_LATEST|


#. Restart the service:

    .. include:: ../../_templates/installations/basic/elastic/common/enable_elasticsearch.rst

#. Start the newly-upgraded node and confirm that it joins the cluster by checking the log file or by submitting a ``_cat/nodes`` request:

    .. code-block:: bash

      curl -X GET "https://127.0.0.1:9200/_cat/nodes" -u <username>:<password> -k

#. Reenable shard allocation:

    .. code-block:: bash

      curl -X PUT "https://127.0.0.1:9200/_cluster/settings" -u <username>:<password> -k -H 'Content-Type: application/json' -d'
      {
        "persistent": {
          "cluster.routing.allocation.enable": "all"
        }
      }
      '

#. Before upgrading the next node, wait for the cluster to finish shard allocation:

    .. code-block:: bash

      curl -X GET "https://127.0.0.1:9200/_cat/health?v" -u <username>:<password> -k

#. Repeat the steps for every Elasticsearch node.


Upgrading Filebeat
------------------

The following steps needs to be run in the Wazuh server or servers in case of Wazuh multi-node cluster. 


#. Upgrade Filebeat:

    .. tabs::

      .. group-tab:: YUM

        .. code-block:: console

          # yum install filebeat-|ELASTICSEARCH_ELK_LATEST|

      .. group-tab:: APT

        .. code-block:: console

          # apt-get install filebeat=|ELASTICSEARCH_ELK_LATEST|

      .. group-tab:: ZYpp

        .. code-block:: console

          # zypper update filebeat-|ELASTICSEARCH_ELK_LATEST|


#. Download the alerts template for Elasticsearch:

    .. code-block:: console

      # curl -so /etc/filebeat/wazuh-template.json https://raw.githubusercontent.com/wazuh/wazuh/v|WAZUH_LATEST|/extensions/elasticsearch/7.x/wazuh-template.json
      # chmod go+r /etc/filebeat/wazuh-template.json


#. Download the Wazuh module for Filebeat:

    .. code-block:: console

      # curl -s https://packages.wazuh.com/4.x/filebeat/wazuh-filebeat-0.1.tar.gz | sudo tar -xvz -C /usr/share/filebeat/module


#. Edit the ``/etc/filebeat/filebeat.yml`` configuration file. Replace ``YOUR_ELASTIC_SERVER_IP`` with the IP address or the hostname of the Elasticsearch server. For example:

    .. code-block:: yaml

      output.elasticsearch.hosts: ['http://YOUR_ELASTIC_SERVER_IP:9200']


#. Restart Filebeat:

    .. include:: ../../_templates/installations/basic/elastic/common/enable_filebeat.rst


Upgrading Kibana
----------------

.. warning::
  Since Wazuh 3.12.0 release, regardless of the Elastic Stack version, the location of the Wazuh Kibana plugin configuration file has been moved from ``/usr/share/kibana/plugins/wazuh/wazuh.yml``, for the version 3.11.x, and from ``/usr/share/kibana/plugins/wazuh/config.yml``, for the version 3.10.x or older, to ``/usr/share/kibana/optimize/wazuh/config/wazuh.yml``.

Copy the Wazuh Kibana plugin configuration file to its new location. This step is not needed for upgrades from 3.12.x to latest:

      .. tabs::

          .. group-tab:: For upgrades from 3.11.x to latest

              Create the new directory and copy the Wazuh Kibana plugin configuration file:

                .. code-block:: console

                  # mkdir -p /usr/share/kibana/optimize/wazuh/config
                  # cp /usr/share/kibana/plugins/wazuh/wazuh.yml /usr/share/kibana/optimize/wazuh/config/wazuh.yml


          .. group-tab:: For upgrades from 3.10.x or older to latest


              Create the new directory and copy the Wazuh Kibana plugin configuration file:

                    .. code-block:: console

                      # mkdir -p /usr/share/kibana/optimize/wazuh/config
                      # cp /usr/share/kibana/plugins/wazuh/config.yml /usr/share/kibana/optimize/wazuh/config/wazuh.yml


              Edit the ``/usr/share/kibana/optimize/wazuh/config/wazuh.yml`` configuration file and add to the end of the file the following default structure to define an Wazuh API entry:

                    .. code-block:: yaml

                      hosts:
                        - <id>:
                           url: http(s)://<api_url>
                           port: <api_port>
                           user: <api_user>
                           password: <api_password>

                    The following values need to be replaced:

                      -  ``<id>``: an arbitrary ID.

                      -  ``<api_url>``: url of the Wazuh API.

                      -  ``<api_port>``: port.

                      -  ``<api_user>``: credentials to authenticate.

                      -  ``<api_password>``: credentials to authenticate.

                    In case of having more Wazuh API entries, each of them must be added manually.

 
#. Replace the value ``user`` by ``username`` and set the username and password as ``wazuh-wui`` in the file ``/usr/share/kibana/optimize/wazuh/config/wazuh.yml``: 

    .. code-block:: yaml
      :emphasize-lines: 5, 6

      hosts:
        - default:
          url: https://localhost
          port: 55000
          username: wazuh-wui
          password: wazuh-wui

#. Remove the Wazuh Kibana plugin:

    .. code-block:: console

      # cd /usr/share/kibana/
      # sudo -u kibana bin/kibana-plugin remove wazuh

#. Upgrade Kibana:

      .. tabs::

        .. group-tab:: YUM

          .. code-block:: console

            # yum install kibana-|ELASTICSEARCH_ELK_LATEST|

        .. group-tab:: APT

          .. code-block:: console

            # apt-get install kibana=|ELASTICSEARCH_ELK_LATEST|

        .. group-tab:: ZYpp

          .. code-block:: console

            # zypper update kibana=|ELASTICSEARCH_ELK_LATEST|

#. Remove generated bundles and the ``wazuh-registry.json`` file:

    .. code-block:: console

      # rm -rf /usr/share/kibana/optimize/bundles
      # rm -f /usr/share/kibana/optimize/wazuh/config/wazuh-registry.json

#. Update file permissions. This will prevent errors when generating new bundles or updating the Wazuh Kibana plugin:

    .. code-block:: console

      # chown -R kibana:kibana /usr/share/kibana/optimize
      # chown -R kibana:kibana /usr/share/kibana/plugins

#. Install the Wazuh Kibana plugin:

    .. code-block:: console

      # cd /usr/share/kibana/
      # sudo -u kibana /usr/share/kibana/bin/kibana-plugin install https://packages.wazuh.com/4.x/ui/kibana/wazuh_kibana-4.0.0_7.9.2-1.zip


#. Update configuration file permissions:

    .. code-block:: console

      # sudo chown kibana:kibana /usr/share/kibana/optimize/wazuh/config/wazuh.yml
      # sudo chmod 600 /usr/share/kibana/optimize/wazuh/config/wazuh.yml

#. For installations on Kibana 7.6.x version and higher, it is recommended to increase the heap size of Kibana to ensure the Kibana's plugins installation:

    .. code-block:: console

      # cat >> /etc/default/kibana << EOF
      NODE_OPTIONS="--max_old_space_size=2048"
      EOF

#. Restart Kibana:

    .. include:: ../../_templates/installations/basic/elastic/common/enable_kibana.rst


#. Once Kibana is accesible, remove the ``wazuh-alerts-3.x-*`` index pattern. Since Wazuh 4.0 it has been replaced by ``wazuh-alerts-*`` , it is necessary to remove the old pattern in order for the new one to take its place.

    .. code-block:: console

      # curl 'https://<kibana_ip>:<kibana_port>/api/saved_objects/index-pattern/wazuh-alerts-3.x-*' -X DELETE  -H 'Content-Type: application/json' -H 'kbn-version: |ELASTICSEARCH_ELK_LATEST|' -k -uelastic:<elastic_password>

    If you have a custom index pattern, be sure to replace it accordingly.

#. Clean the browser's cache and cookies.



Disabling the repository
^^^^^^^^^^^^^^^^^^^^^^^^

It is recommended to disable the Elastic repository to prevent an upgrade to a newest Elastic Stack version due to the possibility of undoing changes with the Wazuh Kibana plugin:


      .. tabs::

        .. group-tab:: YUM

          .. code-block:: console

            # sed -i "s/^enabled=1/enabled=0/" /etc/yum.repos.d/elastic.repo

        .. group-tab:: APT

          .. code-block:: console

            # sed -i "s/^deb/#deb/" /etc/apt/sources.list.d/elastic-7.x.list
            # apt-get update

          Alternatively, the user can set the package state to ``hold``, which will stop updates. It will be still possible to upgrade it manually using ``apt-get install``:

          .. code-block:: console

            # echo "elasticsearch hold" | sudo dpkg --set-selections
            # echo "filebeat hold" | sudo dpkg --set-selections
            # echo "kibana hold" | sudo dpkg --set-selections

        .. group-tab:: ZYpp

          .. code-block:: console

            # sed -i "s/^enabled=1/enabled=0/" /etc/zypp/repos.d/elastic.repo

Next step
---------

The next step consists on :ref:`upgrading the Wazuh agents<upgrading_wazuh_agent>`.
