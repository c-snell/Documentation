source

:   hpe3par\_host.py

hpe3par\_host - Manage HPE 3PAR Host
====================================

Synopsis
--------

-   On HPE 3PAR - Create Host. - Delete Host. - Add Initiator Chap. -
    Remove Initiator Chap. - Add Target Chap. - Remove Target Chap. -
    Add FC Path to Host - Remove FC Path from Host - Add ISCSI Path to
    Host - Remove ISCSI Path from Host

Parameters
----------

<table  border=0 cellpadding=0 class="documentation-table">
            <tr>
        <th class="head"><div class="cell-border">Parameter</div></th>
        <th class="head"><div class="cell-border">Choices/<font color="blue">Defaults</font></div></th>
                    <th class="head" width="100%"><div class="cell-border">Comments</div></th>
    </tr>
                <tr class="return-value-column">
                            <td>
                <div class="outer-elbow-container">
                                            <div class="elbow-key">
                        <b>chap_name</b>
                                                                            </div>
                </div>
            </td>
                            <td>
                <div class="cell-border">
                                                                                                                                                                                        </div>
            </td>
                                                            <td>
                <div class="cell-border">
                                                                                <div>The chap name.
Required with actions add_initiator_chap, add_target_chap
</div>
                                                                                            </div>
            </td>
        </tr>
                            <tr class="return-value-column">
                            <td>
                <div class="outer-elbow-container">
                                            <div class="elbow-key">
                        <b>chap_secret</b>
                                                                            </div>
                </div>
            </td>
                            <td>
                <div class="cell-border">
                                                                                                                                                                                        </div>
            </td>
                                                            <td>
                <div class="cell-border">
                                                                                <div>The chap secret for the host or the target
Required with actions add_initiator_chap, add_target_chap
</div>
                                                                                            </div>
            </td>
        </tr>
                            <tr class="return-value-column">
                            <td>
                <div class="outer-elbow-container">
                                            <div class="elbow-key">
                        <b>chap_secret_hex</b>
                                                                            </div>
                </div>
            </td>
                            <td>
                <div class="cell-border">
                                                                                                                                                                                                    <ul><b>Choices:</b>
                                                                                                                                                                                <li>no</li>
                                                                                                                                                                                                                    <li>yes</li>
                                                                                            </ul>
                                                                                        </div>
            </td>
                                                            <td>
                <div class="cell-border">
                                                                                <div>If true, then chapSecret is treated as Hex.</div>
                                                                                            </div>
            </td>
        </tr>
                            <tr class="return-value-column">
                            <td>
                <div class="outer-elbow-container">
                                            <div class="elbow-key">
                        <b>force_path_removal</b>
                                                                            </div>
                </div>
            </td>
                            <td>
                <div class="cell-border">
                                                                                                                                                                                                    <ul><b>Choices:</b>
                                                                                                                                                                                <li>no</li>
                                                                                                                                                                                                                    <li>yes</li>
                                                                                            </ul>
                                                                                        </div>
            </td>
                                                            <td>
                <div class="cell-border">
                                                                                <div>If true, remove WWN(s) or iSCS<em>s</em> even if there are VLUNs that are exported to the host.
</div>
                                                                                            </div>
            </td>
        </tr>
                            <tr class="return-value-column">
                            <td>
                <div class="outer-elbow-container">
                                            <div class="elbow-key">
                        <b>host_domain</b>
                                                                            </div>
                </div>
            </td>
                            <td>
                <div class="cell-border">
                                                                                                                                                                                        </div>
            </td>
                                                            <td>
                <div class="cell-border">
                                                                                <div>Create the host in the specified domain, or in the default domain, if unspecified
</div>
                                                                                            </div>
            </td>
        </tr>
                            <tr class="return-value-column">
                            <td>
                <div class="outer-elbow-container">
                                            <div class="elbow-key">
                        <b>host_fc_wwns</b>
                                                                            </div>
                </div>
            </td>
                            <td>
                <div class="cell-border">
                                                                                                                                                                                        </div>
            </td>
                                                            <td>
                <div class="cell-border">
                                                                                <div>Set one or more WWNs for the host.
Required with action add_fc_path_to_host, remove_fc_path_from_host
</div>
                                                                                            </div>
            </td>
        </tr>
                            <tr class="return-value-column">
                            <td>
                <div class="outer-elbow-container">
                                            <div class="elbow-key">
                        <b>host_iscsi_names</b>
                                                                            </div>
                </div>
            </td>
                            <td>
                <div class="cell-border">
                                                                                                                                                                                        </div>
            </td>
                                                            <td>
                <div class="cell-border">
                                                                                <div>Set one or more iSCSI names for the host.
Required with action add_iscsi_path_to_host, remove_iscsi_path_from_host
</div>
                                                                                            </div>
            </td>
        </tr>
                            <tr class="return-value-column">
                            <td>
                <div class="outer-elbow-container">
                                            <div class="elbow-key">
                        <b>host_name</b>
                        <br/><div style="font-size: small; color: red">required</div>                                                    </div>
                </div>
            </td>
                            <td>
                <div class="cell-border">
                                                                                                                                                                                        </div>
            </td>
                                                            <td>
                <div class="cell-border">
                                                                                <div>Name of the Host.</div>
                                                                                            </div>
            </td>
        </tr>
                            <tr class="return-value-column">
                            <td>
                <div class="outer-elbow-container">
                                            <div class="elbow-key">
                        <b>host_new_name</b>
                        <br/><div style="font-size: small; color: red">required</div>                                                    </div>
                </div>
            </td>
                            <td>
                <div class="cell-border">
                                                                                                                                                                                        </div>
            </td>
                                                            <td>
                <div class="cell-border">
                                                                                <div>New name of the Host.</div>
                                                                                            </div>
            </td>
        </tr>
                            <tr class="return-value-column">
                            <td>
                <div class="outer-elbow-container">
                                            <div class="elbow-key">
                        <b>host_persona</b>
                                                                            </div>
                </div>
            </td>
                            <td>
                <div class="cell-border">
                                                                                                                                                <ul><b>Choices:</b>
                                                                                                                                                                                <li>GENERIC</li>
                                                                                                                                                                                                                    <li>GENERIC_ALUA</li>
                                                                                                                                                                                                                    <li>GENERIC_LEGACY</li>
                                                                                                                                                                                                                    <li>HPUX_LEGACY</li>
                                                                                                                                                                                                                    <li>AIX_LEGACY</li>
                                                                                                                                                                                                                    <li>EGENERA</li>
                                                                                                                                                                                                                    <li>ONTAP_LEGACY</li>
                                                                                                                                                                                                                    <li>VMWARE</li>
                                                                                                                                                                                                                    <li>OPENVMS</li>
                                                                                                                                                                                                                    <li>HPUX</li>
                                                                                                                                                                                                                    <li>WINDOWS_SERVER</li>
                                                                                            </ul>
                                                                                        </div>
            </td>
                                                            <td>
                <div class="cell-border">
                                                                                <div>ID of the persona to assign to the host. Uses the default persona unless you specify the host persona.
</div>
                                                                                            </div>
            </td>
        </tr>
                            <tr class="return-value-column">
                            <td>
                <div class="outer-elbow-container">
                                            <div class="elbow-key">
                        <b>state</b>
                        <br/><div style="font-size: small; color: red">required</div>                                                    </div>
                </div>
            </td>
                            <td>
                <div class="cell-border">
                                                                                                                                                <ul><b>Choices:</b>
                                                                                                                                                                                <li>present</li>
                                                                                                                                                                                                                    <li>absent</li>
                                                                                                                                                                                                                    <li>modify</li>
                                                                                                                                                                                                                    <li>add_initiator_chap</li>
                                                                                                                                                                                                                    <li>remove_initiator_chap</li>
                                                                                                                                                                                                                    <li>add_target_chap</li>
                                                                                                                                                                                                                    <li>remove_target_chap</li>
                                                                                                                                                                                                                    <li>add_fc_path_to_host</li>
                                                                                                                                                                                                                    <li>remove_fc_path_from_host</li>
                                                                                                                                                                                                                    <li>add_iscsi_path_to_host</li>
                                                                                                                                                                                                                    <li>remove_iscsi_path_from_host</li>
                                                                                            </ul>
                                                                                        </div>
            </td>
                                                            <td>
                <div class="cell-border">
                                                                                <div>Whether the specified Host should exist or not. State also provides actions to add and remove initiator and target chap, add fc/iscsi path to host.
</div>
                                                                                            </div>
            </td>
        </tr>
                            <tr class="return-value-column">
                            <td>
                <div class="outer-elbow-container">
                                            <div class="elbow-key">
                        <b>storage_system_ip</b>
                        <br/><div style="font-size: small; color: red">required</div>                                                    </div>
                </div>
            </td>
                            <td>
                <div class="cell-border">
                                                                                                                                                                                        </div>
            </td>
                                                            <td>
                <div class="cell-border">
                                                                                <div>The storage system IP address.</div>
                                                                                            </div>
            </td>
        </tr>
                            <tr class="return-value-column">
                            <td>
                <div class="outer-elbow-container">
                                            <div class="elbow-key">
                        <b>storage_system_password</b>
                        <br/><div style="font-size: small; color: red">required</div>                                                    </div>
                </div>
            </td>
                            <td>
                <div class="cell-border">
                                                                                                                                                                                        </div>
            </td>
                                                            <td>
                <div class="cell-border">
                                                                                <div>The storage system password.</div>
                                                                                            </div>
            </td>
        </tr>
                            <tr class="return-value-column">
                            <td>
                <div class="outer-elbow-container">
                                            <div class="elbow-key">
                        <b>storage_system_username</b>
                        <br/><div style="font-size: small; color: red">required</div>                                                    </div>
                </div>
            </td>
                            <td>
                <div class="cell-border">
                                                                                                                                                                                        </div>
            </td>
                                                            <td>
                <div class="cell-border">
                                                                                <div>The storage system user name.</div>
                                                                                            </div>
            </td>
        </tr>
                    </table>
<br/>
Examples
--------

```yml
- name: Create Host "{{ host_name }}"
  hpe3par_host:
    storage_system_ip="{{ storage_system_ip }}"
    storage_system_username="{{ storage_system_username }}"
    storage_system_password="{{ storage_system_password }}"
    state=present
    host_name="{{ host_name }}"

- name: Modify Host "{{ host_name }}"
  hpe3par_host:
    storage_system_ip="{{ storage_system_ip }}"
    storage_system_username="{{ storage_system_username }}"
    storage_system_password="{{ storage_system_password }}"
    state=modify
    host_name="{{ host_name }}"
    host_new_name="{{ host_new_name }}"

- name: Delete Host "{{ new_name }}"
  hpe3par_host:
    storage_system_ip="{{ storage_system_ip }}"
    storage_system_username="{{ storage_system_username }}"
    storage_system_password="{{ storage_system_password }}"
    state=absent
    host_name="{{ host_new_name }}"
```
