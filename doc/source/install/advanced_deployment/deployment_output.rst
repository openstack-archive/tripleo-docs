The output from ``ansible-playbook`` will then begin to appear in the console
and will be updated periodically as more tasks are applied.

When ansible is finished a play recap will be shown, and the usual overcloudrc
details will then be displayed. The following is an example of the end of the
output from a successful deployment::

    PLAY RECAP ****************************************************************
    compute-0                  : ok=134  changed=48   unreachable=0    failed=0
    openstack-0                : ok=164  changed=28   unreachable=0    failed=1
    openstack-1                : ok=160  changed=28   unreachable=0    failed=0
    openstack-2                : ok=160  changed=28   unreachable=0    failed=0
    pacemaker-0                : ok=138  changed=30   unreachable=0    failed=0
    pacemaker-1                : ok=138  changed=30   unreachable=0    failed=0
    pacemaker-2                : ok=138  changed=30   unreachable=0    failed=0
    undercloud                 : ok=2    changed=0    unreachable=0    failed=0

    Overcloud configuration completed.
    Overcloud Endpoint: http://192.168.24.8:5000/
    Overcloud rc file: /home/stack/overcloudrc
    Overcloud Deployed

When a failure happens, the deployment will stop and the error will be shown.

Review the ``PLAY RECAP`` which will show each host that is part of the
overcloud and the grouped count of each task status.
