git init
git branch -M main
git remote add origin "주소"
git remote -v                                   //check 용도
git config --global user.email "이메일"
git config --global user.name "이름"

타 기기에서 엑세스

git clone "주소"
git config --global user.email "이메일"
git config --global user.name "이름"

각자 자신의 파일만 변경할 것!!!
변경사항 보내기
git add .    
git commit -m "message"
git push origin master

변경사항 가져오기
git add .    
git commit -m "message"
git pull

kenney