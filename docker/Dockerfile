from busybox

ADD export/hard_work /app/hard_work
RUN echo ">>>>> I'm doing intensive work based on hard_work now! <<<<<"

ADD export /app
RUN echo ">>>>> The rest was added and invalidated the cache. <<<<<"
