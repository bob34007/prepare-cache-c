# prepare-cache-c
Prepare -> execute->close is not used in the MYSQL C API. The client returns the last STMT handle when it calls close. The STMT cache is deprecated by initialization time (consider using LRU linked lists later) to improve C API call performance.
