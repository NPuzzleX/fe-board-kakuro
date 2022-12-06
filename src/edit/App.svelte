<script lang="ts">
  import { onMount } from "svelte";
  import { postPuzzle, getPuzzle } from "../helper/backend";
  import html2canvas from "html2canvas"
  import * as iconList from "../helper/iconList";

  let mainDiv: HTMLDivElement;
  let hidden: boolean = false;

  let originalHeight = 1200;
  let originalWidth = 1200;
  let scaling = 1;
  let zoomFactor = 1;

  let cellFontSize = 12;

  $: {
    cellFontSize = (originalHeight > originalWidth ? (originalHeight * scaling/rowNum) : (originalWidth * scaling/colNum))/2;
  }

  let mainAreaHeight: number;
  let mainAreaWidth: number;
  let optionHeight: number;
  let fitPage:boolean = true;
  let optionHidden: boolean = false;

  $: isOverflowWidth = fitPage ? false : (!mainDiv ? false : (mainDiv.clientWidth < originalWidth*scaling));
  $: isOverflowheight = fitPage ? false : (!mainDiv ? false : (mainDiv.clientHeight < originalHeight*scaling));

  $: {
    if (fitPage) {
      if ((mainAreaWidth)/colNum > (mainAreaHeight - (optionHidden ? 0 : optionHeight))/rowNum) {
        zoomFactor = (mainAreaHeight - (optionHidden ? 0 : optionHeight))/mainAreaHeight;
        scaling = mainAreaHeight/originalHeight*zoomFactor;
      } else {
        zoomFactor = (mainAreaWidth)/mainAreaWidth;
        scaling = mainAreaWidth/originalWidth*zoomFactor;
      }
      cellFontSize = (originalHeight > originalWidth ? (originalHeight * scaling/rowNum) : (originalWidth * scaling/colNum))/2;
    } else {
      if (zoomFactor <= 0) {
        zoomFactor = 1;
      }

      if ((mainAreaWidth - 15)/colNum > (mainAreaHeight - 15)/rowNum) {
        scaling = mainAreaHeight/originalHeight*zoomFactor;
      } else {
        scaling = mainAreaWidth/originalWidth*zoomFactor;
      }
      cellFontSize = (originalHeight > originalWidth ? (originalHeight * scaling/rowNum) : (originalWidth * scaling/colNum))/2;
    }
  }

  $: {
    if (rowNum < 1) {
      rowNum = 1;
    } else if (Math.round(rowNum) != rowNum) {
      rowNum = Math.round(rowNum);
    }
  }

  $: {
    if (colNum < 1) {
      colNum = 1;
    } else if (Math.round(colNum) != colNum) {
      colNum = Math.round(colNum);
    }
  }

  let isMouseDown:boolean = false;
  let isScrolled:boolean = false;

  function mouseMove(e: MouseEvent) {
    if (isMouseDown) {
      e.preventDefault();

      if (Math.abs(e.movementX) + Math.abs(e.movementY) > 2)  {
        mainDiv.scrollTop = mainDiv.scrollTop - e.movementY;
        mainDiv.scrollLeft = mainDiv.scrollLeft - e.movementX;
      }
    }
  }

  function mouseDown(e: MouseEvent) {
    isMouseDown = true;
  }

  function mouseUp() {
    if (isMouseDown) {
      isMouseDown = false;
    }
  }

  function mouseLeave () {
    if (isMouseDown) {
      isMouseDown = false;
    }
    isScrolled = false;
  }

  onMount(async () => {
    const target = document.getElementById("optionBoard");
   
    const observer = new MutationObserver(e => {
      if (e[0].attributeName == "style") {
        if (e[0].target.firstChild.parentElement.style.display == "none") {
          optionHidden = true;
        } else {
          optionHidden = false;
        }
      }
		})
		
	  observer.observe(target, { attributeFilter: ["style"] });

    if (!sessionStorage.getItem("PuzzleId")) {
      return;
    }

    const tempData = await getPuzzle();
    if (!tempData) {
      console.log("INVALID PUZZLE_ID");
      window.postMessage("INVALID PUZZLE_ID", window.origin);
    }
    
    try {
      Val = tempData.data.map(e => {
        return (e.map(e2 => {
          let a: boardData = {
            v: e2.v,
            dt1: e2.dt1 ? e2.dt1 : "",
            dt2: e2.dt2 ? e2.dt2 : "",
            w: false,
            w1: false,
            w2: false
          }
          return (a);
        }))
      });
    } catch (error) {
      console.log("INVALID PUZZLE_ID");
      window.postMessage("INVALID PUZZLE_ID", window.origin);
    }
    
    colNum = Val.length;
    rowNum = Val[0].length;    
  });

  //------------- PLAY BOARD -------------
  type boardData = {
    v: number,
    /*
      -2: block
      -1: border
      0: empty
      1-9: number
    */
    dt1: string,
    dt2: string,
    w: boolean, // confirmed wrong
    w1: boolean,
    w2: boolean
  }

  let colNum: number = 5;
  let rowNum: number = 5;

  let Val: boardData[][] = [];

  while (Val.length < rowNum) {
    let a: boardData[] = [];
    while (a.length < colNum) {
      a.push({v: -2, dt1: "", dt2: "", w: false, w1: false, w2: false});
    }
    Val.push(a);
  }

  $: {
    while (Val.length > rowNum) {
      Val.pop();
    }

    while (Val.length < rowNum) {
      let a: boardData[] = [];
      while (a.length < colNum) {
        a.push({v: -2, dt1: "", dt2: "", w: false, w1: false, w2: false});
      }
      Val.push(a);
    }

    Val.forEach(e => {
      while (e.length > colNum) {
        e.pop();
      }

      while (e.length < colNum) {
        e.push({v: -2, dt1: "", dt2: "", w: false, w1: false, w2: false});
      }
    })

    for(let i = 0; i < colNum; i++) {
      checkSurroundingCell(rowNum - 1, i);
    }

    for(let i = 0; i < rowNum; i++) {
      checkSurroundingCell(i, colNum - 1);
    }

    originalWidth = originalHeight*colNum/rowNum;
    forceRefresh();
  }

  function forceRefresh() {
    Val = Val;
  }

  function mainPanelClick(i: number, j: number) {
    if (isScrolled) {
      isScrolled = false;
      return;
    }

    if (Val[i][j].v == 0) {
      Val[i][j].v = -2;
      Val[i][j].dt1 = "";
      Val[i][j].dt2 = "";
    } else  {
      Val[i][j].v = 0;
    }

    checkSurroundingCell(i, j);
    if (i > 0) {
      checkSurroundingCell(i - 1, j);
      if (j > 0) {
        checkSurroundingCell(i - 1, j - 1);
      }
      if (j < colNum - 1) {
        checkSurroundingCell(i - 1, j + 1);
      }
    }

    if (i < rowNum - 1) {
      checkSurroundingCell(i + 1, j);
      if (j > 0) {
        checkSurroundingCell(i + 1, j - 1);
      }
      if (j < colNum - 1) {
        checkSurroundingCell(i + 1, j + 1);
      }
    }

    if (j > 0) {
      checkSurroundingCell(i, j - 1);
    }
    if (j < colNum - 1) {
      checkSurroundingCell(i, j + 1);
    }
  }

  function checkSurroundingCell(i: number, j: number) { 
    if (Val[i][j].v == 0) {
      return;
    }

    if (i > 0) {
      if (Val[i - 1][j].v == 0) {
        Val[i][j].v = -1;
        return;
      }
    }
    
    if (i < rowNum - 1) {
      if (Val[i + 1][j].v == 0) {
        Val[i][j].v = -1;
        return;
      }
    }

    if (j > 0) {
      if (Val[i][j - 1].v == 0) {
        Val[i][j].v = -1;
        return;
      }
    }
    
    if (j < colNum - 1) {
      if (Val[i][j + 1].v == 0) {
        Val[i][j].v = -1;
        return;
      }
    }

    Val[i][j].v = -2;
  }

  function clearState() {
    for(let i = 0; i < Val.length; i++) {
      for(let j = 0; j < Val[i].length; j++) {
        Val[i][j].v = -2;
        Val[i][j].w = false;
        Val[i][j].w1 = false;
        Val[i][j].w2 = false;
        Val[i][j].dt1 = "";
        Val[i][j].dt2 = "";
      }
    }
  }

  function validateInput(i: number, j: number, upper: boolean) {
    if (upper) {
      let check = parseInt(Val[i][j].dt1);
      if (isNaN(check)) {
        Val[i][j].dt1 = "";
      }
      /*
      if (check > 45) {
        Val[i][j].dt1 = "";
      }
      */
    } else {
      let check = parseInt(Val[i][j].dt2);
      if (isNaN(check)) {
        Val[i][j].dt2 = "";
      } 
      /* 
      if (check > 45) {
        Val[i][j].dt2 = "";
      }
      */
    }
  }

  function selectAllText(e: Event) {
    window.getSelection().selectAllChildren((e.target as Element))
  }  
  //------------- SAVING + THUMBNAIL GENERATOR -------------
  let thumbnailTarget: HTMLDivElement;
  const maxThumbnailSize: number = 400;
  let checking: boolean = false;

  async function check() {
    if (!checking) {
      const minVal: number[] = [0, 1, 3, 6, 10, 15, 21, 28, 36, 45];
      const maxVal: number[] = [0, 9, 17, 24, 30, 35, 39, 42, 44, 45];

      checking = true;
      let clear = true;

      for (let i = 0; i < Val.length; i++) {
        for (let j = 0; j < Val[i].length; j++) {
          if (Val[i][j].v == -1) {
            if ((Val[i][j].dt1 != "") && (isNaN(parseInt(Val[i][j].dt1)))) {
              Val[i][j].w1 = true;
              clear = false;
            }
            if ((Val[i][j].dt2 != "") && (isNaN(parseInt(Val[i][j].dt2)))) {
              Val[i][j].w2 = true;
              clear = false;
            }
            if (parseInt(Val[i][j].dt1) > 45) {
              Val[i][j].w1 = true;
              clear = false;
            }
            if (parseInt(Val[i][j].dt2) > 45) {
              Val[i][j].w2 = true;
              clear = false;
            }              

            let empty: number[] = [0, 0, 0, 0];
            for(let x = i - 1; x > 0; x--) {
              if (Val[x][j].v == 0) {empty[0]++;} else {break;}
            }
            if (empty[0] > 9) {
              for(let x = i - 1; x > 0; x--) {
                if (Val[x][j].v == 0) {Val[x][j].w = true;} else {break;}
              }
              clear = false;
            }

            for(let x = i + 1; x < rowNum; x++) {
              if (Val[x][j].v == 0) {empty[2]++;} else {break;}
            }
            if (empty[2] > 9) {
              for(let x = i + 1; x < rowNum; x++) {
                if (Val[x][j].v == 0) {Val[x][j].w = true;} else {break;}
              }
              clear = false;
            }

            for(let y = j - 1; y > 0; y--) {
              if (Val[i][y].v == 0) {empty[3]++;} else {break;}
            }
            if (empty[3] > 9) {
              for(let y = j - 1; y > 0; y--) {
                if (Val[i][y].v == 0) {Val[i][y].w = true;} else {break;}
              }
              clear = false;
            }

            for(let y = j + 1; y < colNum; y++) {
              if (Val[i][y].v == 0) {empty[1]++;} else {break;}
            }
            if (empty[1] > 9) {
              for(let y = j + 1; y < colNum; y++) {
                if (Val[i][y].v == 0) {Val[i][y].w = true;} else {break;}
              }
              clear = false;
            }

            if (Val[i][j].dt1 != "") {
              if (Math.max(empty[0], empty[1]) == 0) {
                Val[i][j].dt1 = "";
              } else if ((empty[0] <= 9) && (empty[1] <= 9)) {
                const valNum = parseInt(Val[i][j].dt1);
                if (empty[0] == 0) {
                  if ((valNum < minVal[empty[1]]) || (valNum > maxVal[empty[1]])) {
                    Val[i][j].w1 = true;
                    clear = false;
                  }
                } else if (empty[1] == 0) {
                  if ((valNum < minVal[empty[0]]) || (valNum > maxVal[empty[0]])) {
                    Val[i][j].w1 = true;
                    clear = false;
                  }
                } else if ((valNum < minVal[Math.min(empty[0], empty[1])]) || (valNum > maxVal[Math.max(empty[0], empty[1])])) {
                  Val[i][j].w1 = true;
                  clear = false;
                }
              }
            }

            if (Val[i][j].dt2 != "") {
              if (Math.max(empty[2], empty[2]) == 0) {
                Val[i][j].dt2 = "";
              } else if ((empty[2] <= 9) && (empty[3] <= 9)) {
                const valNum = parseInt(Val[i][j].dt2);
                if (empty[2] == 0) {
                  if ((valNum < minVal[empty[3]]) || (valNum > maxVal[empty[3]])) {
                    Val[i][j].w2 = true;
                    clear = false;
                  }
                } else if (empty[3] == 0) {
                  if ((valNum < minVal[empty[2]]) || (valNum > maxVal[empty[2]])) {
                    Val[i][j].w2 = true;
                    clear = false;
                  }
                } else if ((valNum < minVal[Math.min(empty[2], empty[3])]) || (valNum > maxVal[Math.max(empty[2], empty[3])])) {
                  Val[i][j].w2 = true;
                  clear = false;
                }
              }
            }
          }
        }
      }

      if (!clear) {
        setTimeout(() => {
          checking = false;
          for (let i = 0; i < Val.length; i++) {
            for (let j = 0; j < Val[i].length; j++) {
              Val[i][j].w = false;
              Val[i][j].w1 = false;
              Val[i][j].w2 = false;
            }
          }
        }, 2000);
      } else {
        let renderedWidth = maxThumbnailSize;
        let renderedHeight = renderedWidth*originalHeight/originalWidth;

        if (renderedHeight > maxThumbnailSize) {
          renderedHeight = maxThumbnailSize;
          renderedWidth = renderedHeight*originalWidth/originalHeight;
        }

        let cvs = await html2canvas(thumbnailTarget, {
          scale: renderedWidth/thumbnailTarget.clientWidth
        });

        if (await postPuzzle({
          game: Val.map(e => {
            return e.map(e2 => {
              let dt: any = {
                v: 0,
                dt1: undefined,
                dt2: undefined
              };

              dt.v = e2.v;
              if (dt.v == -1) {
                if (e2.dt1 != "") {
                  dt.dt1 = e2.dt1
                } else {
                  delete dt.dt1;
                }
                if (e2.dt2 != "") {
                  dt.dt2 = e2.dt2
                } else {
                  delete dt.dt2;
                }
              } else {
                delete dt.dt1;
                delete dt.dt2;
              }
              return (dt);
            });
          }),
          thumbnail: cvs.toDataURL("image/webp", 0.5),
          },
          hidden)) {
          console.log("UPLAODED NEW PUZZLE");
          window.postMessage("UPLOADED NEW PUZZLE", window.origin);
        }
      }
    }
  }
</script>

<div id="mainArea" bind:clientWidth={mainAreaWidth} bind:clientHeight={mainAreaHeight}>
  <div id="optionBoard" bind:clientHeight={optionHeight}>
    <div class="optionH">
      <div class="option">
        <div class="icon">{@html iconList.colIco}</div>
        <input type=number style="width: 4em;" bind:value={colNum} min="1">
        <div class="icon">{@html iconList.rowIco}</div>
        <input type=number style="width: 4em;" bind:value={rowNum} min="1">
        <input type=checkbox bind:checked={hidden}>
        <div class="icon hiddenIcon">{@html iconList.hiddenIco}</div>Hidden
        <button on:click={clearState} title="Clear"><div class="icon newIcon">{@html iconList.newIco}</div><p>Clear</p></button>
        <button on:click={check} title="Save"><div class="icon">{@html iconList.saveIco}</div><p>Upload</p></button>
      </div>
      <div class="option" style="padding-left: 5px; border-left: solid 3px black">
        <button on:click={() => {zoomFactor >= 2 ? zoomFactor-- : zoomFactor /= 2; fitPage = false;}} title="Zoom out"><div class="icon">{@html iconList.magMinIco}</div><p>Zoom Out</p></button>
        <input type=number on:input={() => {fitPage = false;}} style="width: 4em;" bind:value={zoomFactor} min="0">
        <button on:click={() => {zoomFactor >= 1 ? zoomFactor++ : zoomFactor *= 2; fitPage = false;}} title="Zoom in"><div class="icon">{@html iconList.magPlusIco}</div><p>Zoom In</p></button>
        <button on:click={() => {fitPage = true;}} title="Fit to page"><div class="icon">{@html iconList.maxIco}</div><p>Fit Page</p></button>
        <button on:click={() => {window.postMessage("TRIGGER FULLSCREEN", window.origin);}} title="Full Screen"><div class="icon">{@html iconList.enlargeIco}</div><p>Full Screen</p></button>
      </div>
    </div>
    <div class="option">
      <h3>Use double click!</h3>
    </div>        
  </div>    
  <div id="boardContainer" bind:this={mainDiv}
    on:scroll={() => {isScrolled = true;}}
    style="display: {isOverflowWidth ? "unset" : "flex"}; padding-left: { (isOverflowWidth ? 20 : 0)}px; padding-top: { (isOverflowheight ? 20 : 0)}px; overflow: {fitPage ? "hidden" : "scroll"};"
  >
    <div id="boardArea" bind:this={thumbnailTarget} style="height: {originalHeight*scaling + (isOverflowheight ? 20 : 0)}px; width: {originalWidth*scaling + (isOverflowWidth ? 20 : 0)}px;">
      <div id="mainBoard" class="kakuro"
        style="z-index: 1; opacity: 1.0; grid-template-columns: repeat({colNum}, 1fr); grid-template-rows: repeat({rowNum}, 1fr); height: {originalHeight*scaling}px; width: {originalWidth*scaling}px; font-size: {cellFontSize}px;"
        on:mousemove={mouseMove}
        on:mouseup={mouseUp}
        on:mousedown={mouseDown}
        on:mouseleave={mouseLeave}
      >
      {#each Val as box, i}
        {#each box as e, j}
          <div id={i.toString()} 
            class="boardCell kakuro" class:border={e.v == -1} class:empty={e.v == 0} class:wrong={e.w && (e.v == 0)}
            on:dblclick ={() => mainPanelClick(i, j)}
          >
            {#if e.v == -1}
            <svg xmlns='http://www.w3.org/2000/svg' version='1.1' preserveAspectRatio='none' viewBox='0 0 10 10' style="position: absolute; height: {originalHeight*scaling/rowNum}px; width: {originalWidth*scaling/colNum}px;">
              <line x1='0' y1='0' x2='10' y2='10' style='stroke-width:0.1px'/>
            </svg>

            <div class="cir1" class:empty={!e.w1} class:wrong={e.w1}>
              <div
                class="borderInner"
                on:focusout={() => {validateInput(i, j, true)}}
                on:focusin={(e2) => {selectAllText(e2)}}
                contenteditable = "true"
                bind:innerHTML={e.dt1}
              >
              </div>              
            </div>
            <div class="cir2" class:empty={!e.w2} class:wrong={e.w2}>
              <div
                class="borderInner"
                on:focusout={() => {validateInput(i, j, true)}}
                on:focusin={(e2) => {selectAllText(e2)}}
                contenteditable = "true"
                bind:innerHTML={e.dt2}
              >
              </div>              
            </div>
            {/if}
          </div>
        {/each}
      {/each}
      </div>
    </div>
  </div>
</div>

<style>
#mainArea {
  display: flex;
  flex-direction: column;
  height: 100%;
  width: 100%;
}

#boardContainer {
  justify-content: center;
}

#mainBoard {
  position: absolute;
  top: 0px;
  left: 0px;
  display: grid;
  border-width: 5px;
  border-style: solid;
}

#mainBoard.kakuro {
  gap: 3px;
}

#boardArea {
  border-radius: 10px;
  position: relative;
  overflow: visible;
}

.boardCell {
  cursor: pointer;
  display: flex;
  justify-content: center;
  align-items: center;
  font-size: inherit;
  border-width: 2px;
  border-style: solid;
}

.boardCell.kakuro {
  border-width: 1px;
  border-style: solid;
}

.boardCell.kakuro.border {
  display: unset;
  font-size: 0.45em;
}

.boardCell.kakuro.border div { 
  height: 50%;
  width: 50%;
  border-radius: 50%;
  text-align: center;
  display: flex;
  justify-content: center;
  align-items: center;
}

.boardCell.kakuro.border div.cir1 {
  position: relative;
  left: 46%;
  top: 4%;
} 

.boardCell.kakuro.border div.cir2 {
  position: relative;
  bottom: 4%;
  left: 4%;
} 

.borderInner {
  background-color: transparent;
  color: inherit;
}

/*   MISC  */
#optionBoard {
  display: flex;
  flex-direction: column;
  justify-content: center;
  position: sticky;
}

.optionH {
  display: flex;
  column-gap: 5px;
  align-items: baseline;
  justify-content: center;
}

.option {
  display: flex;
  column-gap: 5px;
  align-items: baseline;
  justify-content: center;
}

.option button {
  padding: 0.5em;
}

/* SVG CONTROL */
button {
  display: flex;
  flex-direction: column;
  align-items: center;
  row-gap: 2px;
}

h3 {
  margin-top: 3px;
  font-weight: bold;
  color: red;
}
</style>