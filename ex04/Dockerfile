# Nginx 기본 동작
# nginx 시작
# 워커 프로세스 생성
# 자기 자신(메인 프로세스)은 종료
# 워커는 백그라운드에서 계속 실행
# Docker에서는 “foreground process”가 필수

FROM nginx:1.25.3
CMD ["nginx", "-g", "daemon off;"] 