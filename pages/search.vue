<template>
  <NuxtLayout name="search">
    <input type="text" v-model="searchQuery" ref="searchInput" class="search" placeholder="Search" />
    <div style="display: none">
      Search Results for "{{ formattedSearchQuery }}"
    </div>
    <a class="gotoTop" style="opacity: 0; pointer-events: none" id="gotoTop" href="#top"></a>
    <div id="top"></div>
    <ul class="searchResultList">
      <li class="searchResults" :class="
        (index === 0 ? 'bigResult' : '') +
        (searchQuery === 'flip' ? ' flip' : '') +
        (searchQuery === 'waradu' ? ' gradient' : '') +
        (searchQuery === 'resize' ? ' resize' : ' noresize') +
        (searchQuery === 'float' ? ' float' : '') +
        (searchQuery === 'white' ? ' white' : '')
      " v-for="(result, index) in searchResults" :key="index">
        <div class="searchResult">
          <img class="searchResultCover" :src="result.coverURL" />
          <div class="searchResultName">{{ result.name }}</div>
          <div class="searchResultLenght">{{ result.lenght }}</div>
          <div class="searchResultArtist">{{ result.artist }}</div>
          <div class="searchResultPlay" v-on:click="playMusic(result.name)">
            <img src="/svg/bold/play.svg" class="searchResultIMG" />
          </div>
        </div>
      </li>
    </ul>
  </NuxtLayout>
</template>

<script>
// const ytdl = require('ytdl-core');

const itunes = "https://itunes.apple.com/search?";

class Song {
  constructor(json) {
    if ("results" in json) {
      try {
        json = json["results"][0];
      } catch { }
    }
    this.kind = json["kind"];
    this.artistName = json["artistName"];
    this.collectionName = json["collectionName"];
    this.trackName = json["trackName"];
    this.artistViewUrl = json["artistViewUrl"];
    this.collectionViewUrl = json["collectionViewUrl"];
    this.trackViewUrl = json["trackViewUrl"];
    this.image = json["artworkUrl100"];
    this.releaseDate = json["releaseDate"];
    this.collectionExplicitness =
      json["collectionExplicitness"] == "notExplicit" ? false : true;
    this.trackExplicitness =
      json["trackExplicitness"] == "notExplicit" ? false : true;
    this.discCount = json["discCount"];
    this.discNumber = json["discNumber"];
    this.trackCount = json["trackCount"];
    this.trackTimeMillis = json["trackTimeMillis"];
    this.country = json["country"];
    this.primaryGenreName = json["primaryGenreName"];
    this.isStreamable = json["isStreamable"].toString();
    this.artistId = json["artistId"];
    this.collectionId = json["collectionId"];
    this.trackId = json["trackId"];
  }

  getImage() {
    return this.image;
  }

  getName() {
    return this.trackName;
  }

  getArtistName() {
    return this.artistName;
  }

  getCountry() {
    return this.country;
  }

  getTrackViewUrl() {
    return this.trackViewUrl;
  }

  isStreamable() {
    return this.isStreamable;
  }

  getCollectionName() {
    return this.collectionName;
  }

  getResizedImage(size) {
    return resizeImage(this.image, size);
  }

  getIds() {
    return [this.trackId, this.collectionId, this.artistId];
  }

  getLength() {
    return this.trackTimeMillis;
  }

  getLengthNormal() {
    const totalSeconds = Math.floor(this.trackTimeMillis / 1000);
    const minutes = Math.floor(totalSeconds / 60);
    const seconds = totalSeconds % 60;
    return `${minutes}:${seconds.toString().padStart(2, "0")}`;
  }

  isExplicit() {
    return this.trackExplicitness;
  }

  searchForSongName(country = "CH", limit = 50, explicit = true) {
    return get(this.trackName, country, limit, explicit);
  }

  videoURL() { }
}

function resizeImage(url, size) {
  url = url.replace("100x100bb.jpg", `${size}x${size}bb.jpg`);
  url = url.replace("60x60bb.jpg", `${size}x${size}bb.jpg`);
  url = url.replace("30x30bb.jpg", `${size}x${size}bb.jpg`);
  return url;
}

var oldTerm = "";

async function get(term, country = "CH", limit = 50, explicit = true) {
  term = term.replace(/\s+/g, "+");

  const apiUrl = `${itunes}term=${term}&media=music&entity=song&country=${country}&limit=${limit}&explicit=${explicit ? "Yes" : "No"
    }&attribute=genreIndex`;

  return fetch(apiUrl)
    .then((response) => response.json())
    .then((data) => {
      const songList = [];
      data.results.forEach((item) => {
        var song = new Song(item);
        songList.push(song);
      });
      return songList;
    });
}

async function playAudio(path) {
  const assetUrl = path;

  const audio = document.getElementById("media");
  audio.innerHTML = "";
  const source = document.createElement("source");
  source.type = "audio/mp3"; // ODER AU video/mp4 wännds nid schafsch zu mp3 konventiere
  source.src = assetUrl;
  audio.appendChild(source);
  audio.load();
  audio.play();
}

async function downloadVideoAsMP3(videoUrl) {
  try {
    const info = await ytdl.getInfo(videoUrl);
    const audioFormat = ytdl.chooseFormat(info.formats, { filter: 'audioonly' });

    // Provide a filename for the downloaded MP3 file
    const filename = 'audio.mp3';

    // Start the download
    ytdl(videoUrl, { format: audioFormat })
      .pipe(fs.createWriteStream(filename))
      .on('finish', () => {
        console.log('Download completed!');
      })
      .on('error', (error) => {
        console.error('Download failed:', error);
      });
  } catch (error) {
    console.error('Failed to download the video:', error);
  }
}


export default {
  data() {
    console;
    return {
      searchQuery: this.$route.query.q || "",
      searchResults: [],
    };
  },
  created() {
    this.search();
  },
  watch: {
    "$route.query.q": function () {
      this.searchQuery = this.$route.query.q || "";
      this.search();
    },
  },
  mounted() {
    this.$nextTick(() => {
      this.$refs.searchInput.focus();
    });
  },
  computed: {
    formattedSearchQuery() {
      get(this.searchQuery).then((songList) => {
        this.searchResults = [];
        songList.forEach((item, index) => {
          if (index == 0) {
            this.searchResults.push({
              name:
                item.getName().length > 12
                  ? item.getName().slice(0, 12).at(-1) == " "
                    ? item.getName().slice(0, 11) + "..."
                    : item.getName().slice(0, 12) + "..."
                  : item.getName(),
              artist:
                item.getArtistName().length > 30
                  ? item.getArtistName().slice(0, 30).at(-1) == " "
                    ? item.getArtistName().slice(0, 29) + "..."
                    : item.getArtistName().slice(0, 30) + "..."
                  : item.getArtistName(),
              lenght: item.getLengthNormal(),
              coverURL: item.getResizedImage(512),
            });
          } else {
            this.searchResults.push({
              name:
                item.getName().length > 20
                  ? item.getName().slice(0, 20).at(-1) == " "
                    ? item.getName().slice(0, 19) + "..."
                    : item.getName().slice(0, 20) + "..."
                  : item.getName(),
              artist:
                item.getArtistName().length > 30
                  ? item.getArtistName().slice(0, 30).at(-1) == " "
                    ? item.getArtistName().slice(0, 29) + "..."
                    : item.getArtistName().slice(0, 30) + "..."
                  : item.getArtistName(),
              lenght: item.getLengthNormal(),
              coverURL: item.getResizedImage(512),
            });
          }
        });
      });
      return this.searchQuery ? this.searchQuery : "";
    },
  },
  methods: {
    search() { },
    async playMusic(query) {
      const url = `https://www.googleapis.com/youtube/v3/search?key=AIzaSyBJ-mQ4fiKtnkMrEZBpCuzlXzIqvtmTsGc&type=video&part=snippet&q=${query}`;
      const response = await fetch(url);
      var data = await response.json();

      data = data.items[0];
      this.searchResults = [
        {
          name: data.snippet.title,
          artist: data.snippet.channelTitle,
          videoId: data.id.videoId,
          coverURL: data.snippet.thumbnails.high.url,
        },
      ];

      console.log(videoId)

      downloadVideoAsMP3(videoId)

      const audioPlayed = new CustomEvent("customPlayAudio", {
        bubbles: true,
        cancelable: true,
        detail: {
          image: music.imageURL, // replce natürli mit de richtige sache
          name: music.trackName,
          artist: music.artistName,
        },
      });
      document.dispatchEvent(audioPlayed);
    },
  },
};
</script>

<style>
@import "~/css/search.css";
</style>
