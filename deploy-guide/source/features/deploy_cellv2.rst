Deploy an additional nova cell v2
=================================

.. warning::
   Multi cell support is only supported in Stein and later versions.

The different sections in this guide assume that you are ready to deploy a new
overcloud, or already have installed an overcloud (min Stein release).

.. note::

   Starting with CentOS 8 and the TripleO Stein release, podman is the CONTAINERCLI
   to be used in the following steps.

The minimum requirement for having multiple cells is to have a central OpenStack
controller cluster running all controller services. Additional cells will
have cell controllers running the cell DB, cell MQ and a nova cell conductor
service. In addition there are 1..n compute nodes. The central nova conductor
service acts as a super conductor of the whole environment.

For more details on the cells v2 layout check `Cells Layout (v2)
<https://docs.openstack.org/nova/latest/user/cellsv2-layout.html>`_

.. toctree::

   deploy_cellv2_basic.rst
   deploy_cellv2_advanced.rst
   deploy_cellv2_routed.rst
   deploy_cellv2_additional.rst
   deploy_cellv2_manage_cell.rst
