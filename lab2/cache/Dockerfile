FROM alpine:3.24
WORKDIR /app
COPY deps.txt .
COPY app.txt .
RUN echo "installing dependencies..." && sleep 15 && cp deps.txt installed.txt
CMD ["cat", "app.txt"]


1.4 builds:
b (app.txt changed): RUN step CACHED = yes, 1.387 s
c (deps.txt changed): RUN step CACHED = no, 17.702 s
d (app.txt changed, wrong order): RUN step CACHED = no, 16.811 s
Earlier results: node images, 1.2 (a1 and b1), and 1.3 (cow:bad 65.4 MB, cow:good 12.9 MB).