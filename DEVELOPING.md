git clone https://github.com/sougat818/insignificantbit.git
git submodule init
git submodule update --remote
cd themes/ananke
git checkout $(git tag | tail -n1 )
open http://localhost:1313/