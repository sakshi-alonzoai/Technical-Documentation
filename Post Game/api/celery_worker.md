# Documentation for `celery_worker.py`

# Celery Worker Startup Script Documentation

## Overview

This script launches a Celery worker that processes tasks from specific queues.  It's designed to be executed directly from the command line to start the worker process, which will then listen for and execute tasks defined within the Celery application (`celery_app`).  The script configures the worker to listen to four designated queues.

## Dependencies

This script relies on the following:

* **Celery:** A distributed task queue.  The specific version will depend on the `celery_app` module's requirements.  Ensure Celery is installed and correctly configured before running this script.  You can typically install it using `pip install celery`.
* **`celery_app.py`:** This module is crucial. It defines the Celery application instance (`celery_app`) and registers all the tasks that this worker will execute.  The script assumes this file is located in the same directory or a location accessible within the Python path.

## Script Breakdown

The script consists primarily of a single `if __name__ == '__main__':` block, ensuring the code within only runs when the script is executed directly, not when imported as a module.


The core functionality lies in the `celery_app.worker_main()` call:

* **`celery_app.worker_main(...)`:** This function, provided by the Celery library, initiates the Celery worker process.  It takes a list of command-line arguments as input, mimicking how you'd start the worker using the `celery worker` command directly.

    * **`['worker']`:** This argument specifies the worker command.
    * **`['--loglevel=info']`:** This sets the logging level to "info". This determines the verbosity of the worker's log output.  Adjust this to `debug`, `warning`, `error`, or `critical` as needed for debugging or reducing log volume.
    * **`['--queues=match_processing,season_processing,data_operations,workflows']`:** This is the most crucial argument. It instructs the worker to listen to the four specified queues: `match_processing`, `season_processing`, `data_operations`, and `workflows`.  Tasks published to these queues will be picked up and processed by this worker instance.


## How to Run

1. **Ensure Dependencies:** Make sure Celery is installed and the `celery_app.py` module (containing your Celery app and task definitions) exists and is accessible.

2. **Execute the Script:** Run the script using the Python interpreter:

   ```bash
   python your_script_name.py
   ```

   Replace `your_script_name.py` with the actual filename of this script.

3. **Monitor the Worker:**  Observe the worker's output for any errors or messages indicating task processing.  The log level specified (`info` in this case) will influence the amount of detail provided.


## Potential Issues and Troubleshooting


* **`celery_app` Module Not Found:** Verify the `celery_app.py` file exists and the script's path allows Python to import it. Correct import statements might require adjustments to your PYTHONPATH.
* **Celery Configuration:** Ensure Celery is correctly configured, particularly broker settings (like RabbitMQ or Redis).  Check the `celery_app.py` for proper broker and backend configuration.
* **Queue Names:** Double-check that the queue names specified (`match_processing`, `season_processing`, `data_operations`, `workflows`) match the queues used when publishing tasks. Mismatches will prevent tasks from being processed.
* **Worker Errors:** Examine the worker's log output for any errors related to task execution.  These logs will help diagnose problems with your Celery tasks or dependencies.

This detailed documentation provides comprehensive instructions and troubleshooting guidance for using the provided Celery worker startup script. Remember to adapt the script and queue names as needed to match your specific Celery application setup.

