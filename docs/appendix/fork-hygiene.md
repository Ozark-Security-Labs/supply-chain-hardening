# Appendix B — Fork hygiene

> Status: placeholder. Ships in **v1.0.0**.

When a transitive dependency starts looking risky — an unresponsive maintainer, dubious provenance, sprawling install footprint — forking and trimming it is sometimes the right move. This appendix documents the decision framework and the mechanical workflow.

Ozark-Security-Labs has implemented this pattern publicly under its `osl-*` prefix convention; this appendix will reference the org's existing internal docs as the runbook source rather than rewriting them. See the org-profile docs for the current procedure.
