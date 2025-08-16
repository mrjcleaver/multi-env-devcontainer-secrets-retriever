This DevContainer system would have provided a secure, multi-environment development with secrets injected from pluggable password managers. It was to use a unified interface that delegates secret resolution to backend-specific adapters, supporting different secret formats and authentication methods per backend.

However, per https://github.com/agenticsorg/community-projects/issues/1#issuecomment-3193745344. https://www.envkey.com/ is a solution that already works.
So let's not build something new, it'd only add work and would need a maintainer.

