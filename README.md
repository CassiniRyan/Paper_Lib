# Paper Lib

This repository is a curated paper library for humanoid robotics, legged locomotion, perception, navigation, whole-body control, teleoperation, motion tracking, and related robot-learning topics.

Papers are stored as PDFs inside topic folders. The compact inventory is maintained in [list.txt](list.txt), while [list.md](list.md) keeps longer reading notes and paper summaries.

## Topic Folders

- `Perceptive`: mapping, elevation maps, LiDAR-inertial odometry, and perception utilities.
- `Locomotion`: perceptive locomotion, parkour, terrain traversal, and agile legged control.
- `Navigation`: navigation, SLAM, traversability, and scene memory.
- `Loco_Manipulation`: humanoid loco-manipulation, interaction, and action-policy papers.
- `Motion_Tracking`: motion tracking, retargeting, mimicking, and motion datasets.
- `Teleoperation_Data`: teleoperation systems and humanoid data-collection pipelines.
- `Motion_Generation`: motion priors, character control, and generative motion methods.
- `Whole_Body_Control`: whole-body control, safety, compliance, actuation, and foundation-control models.
- `Human_Motion_Reconstruction`: human motion recovery and reconstruction.
- `Duplicates`: known duplicate copies kept aside for review.
- `new`: temporary staging area for incoming papers before sorting.

## Maintenance

When adding papers, place incoming PDFs in `new/`, remove duplicates, move unique papers into the closest topic folder, and update [list.txt](list.txt). Keep README focused on repository orientation rather than duplicating the inventory.

Use [tools/pdf_blocks.py](tools/pdf_blocks.py) to control which topic-folder PDFs are pushed or fetched from Git LFS. Set a folder flag in `PUSH_PDFS` or `PULL_PDFS` to `True` for active PDF syncing; leave it `False` to keep those PDFs local-only. The per-folder `INDEX.md` and `papers.txt` files remain tracked either way.

## PDF Sync Control

Edit the flags near the top of [tools/pdf_blocks.py](tools/pdf_blocks.py):

- `PUSH_PDFS`: controls which topic-folder PDFs are staged for GitHub.
- `PULL_PDFS`: controls which Git LFS PDF blocks are downloaded on pull.

Preview the policy without changing the Git index:

```bash
python3 tools/pdf_blocks.py sync
```

Apply the policy:

```bash
python3 tools/pdf_blocks.py sync --apply
```

This always stages `.md` and `.txt` metadata, force-adds PDFs from folders set to `True`, and untracks PDFs from folders set to `False` with `git rm --cached` so the local PDF files are not deleted.

Pull only the enabled LFS PDF blocks:

```bash
python3 tools/pdf_blocks.py pull --apply
```
