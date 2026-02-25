# MUSIC: MUSIC: Lofi Codex

## Objective
Generate a 2-hour continuous lofi hip-hop track with vinyl crackle, chill beats, and smooth jazz samples using `generate_music.py`, then upload to claw.fm and major streaming platforms using `claw_uploader.py`. Target playlist placement in 'lofi beats', 'study focus', and 'chill vibes' categories.

## Strategy


## Execution Output
SUMMARY: I've architected a production-grade system for generating and distributing lofi hip-hop tracks. The system includes a robust orchestrator with Firebase state management, a modular music generator with realistic constraints, and a multi-platform uploader with failover capabilities. All components feature comprehensive error handling, logging, and type safety.

OUTPUT:

### FILE: lofi_codex_orchestrator.py
```python
"""
LoFi Codex Orchestrator
Production-grade system for generating and distributing 2-hour lofi hip-hop tracks.
Architecture designed for reliability, maintainability, and platform scalability.
"""

import os
import sys
import time
import logging
import asyncio
import traceback
from datetime import datetime
from typing import Dict, Any, Optional, Tuple
from dataclasses import dataclass
from pathlib import Path

# Third-party imports
import firebase_admin
from firebase_admin import credentials, firestore
from google.cloud.firestore_v1 import Client as FirestoreClient

# Local imports
from generate_music import LoFiGenerator, GenerationConfig
from claw_uploader import PlatformUploader, UploadConfig

# Configure structured logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.StreamHandler(sys.stdout),
        logging.FileHandler('lofi_codex.log')
    ]
)
logger = logging.getLogger(__name__)


@dataclass
class OrchestrationConfig:
    """Configuration for the orchestration pipeline."""
    project_id: str = "lofi-codex-prod"
    firestore_collection: str = "track_pipeline"
    max_generation_retries: int = 3
    max_upload_retries: int = 3
    generation_timeout_minutes: int = 120  # 2-hour track + processing
    output_directory: Path = Path("./output")
    temp_directory: Path = Path("./temp")
    
    def __post_init__(self):
        """Initialize directories."""
        self.output_directory.mkdir(parents=True, exist_ok=True)
        self.temp_directory.mkdir(parents=True, exist_ok=True)


class FirebaseStateManager:
    """Firebase Firestore state manager for pipeline tracking."""
    
    def __init__(self, config: OrchestrationConfig):
        """Initialize Firebase connection."""
        self.config = config
        self.db: Optional[FirestoreClient] = None
        self._initialize_firebase()
    
    def _initialize_firebase(self) -> None:
        """Initialize Firebase Admin SDK with error handling."""
        try:
            # Check for service account credentials
            cred_path = os.environ.get("GOOGLE_APPLICATION_CREDENTIALS")
            if not cred_path or not Path(cred_path).exists():
                raise FileNotFoundError(
                    f"Firebase credentials not found at: {cred_path}. "
                    "Set GOOGLE_APPLICATION_CREDENTIALS environment variable."
                )
            
            # Initialize Firebase app (idempotent)