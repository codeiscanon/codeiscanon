git clone https://github.com/codeiscanon/codeiscanon.git
git submodule init
git submodule update --remote
cd themes/ananke
git checkout $(git tag | tail -n1 )
open http://localhost:1313/