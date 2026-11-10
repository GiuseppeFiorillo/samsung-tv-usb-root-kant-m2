# Payload

The current payload is generated as ARM assembly by the kernel-stage tools.
UMH variants embed a Bash command and require an explicit listener address.
There is no separately shipped SSH server, credential set, persistent daemon
or bootstrap script in this port.

Upstream's Dropbear-based payload is not copied here. Its functionality and
validation must not be inferred from the upstream directory layout.
